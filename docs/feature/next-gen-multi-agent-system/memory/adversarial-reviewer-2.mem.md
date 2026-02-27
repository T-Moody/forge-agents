# Memory: Adversarial Reviewer 2 (gemini-3-pro-preview perspective)

## Key Facts Learned

1. **SQLite concurrent writes require WAL mode:** When dispatching multiple agents in parallel that INSERT into the same SQLite database (e.g., 3 adversarial reviewers or 4 verifiers), `PRAGMA journal_mode=WAL` and `PRAGMA busy_timeout=5000` are mandatory. Default SQLite locking causes SQLITE_BUSY errors under concurrent writes. This applies to any multi-agent pipeline using a shared SQLite evidence ledger.

2. **`anvil_checks` schema has a `task_id` gap for feature-scoped operations:** The Anvil `anvil_checks` table was designed for per-task verification. When transplanted to a pipeline with feature-scoped operations (e.g., design review before task decomposition), a convention for non-task `task_id` values must be explicitly defined (e.g., `{feature_slug}-design-review`).

3. **Baseline cross-check requires pre-implementation git state:** IDE diagnostics run on current file state only. After implementation, pre-implementation file state is gone from disk. Any "independent baseline verification" mechanism requires either git tags/commits before implementation or file snapshots — it cannot be done by simply re-running diagnostics on modified files.

4. **feature.md FR-1.6 restricts orchestrator to `agent, runSubagent, memory, read_file, list_dir` only:** The design v3 adds `run_in_terminal` to the orchestrator for SQL evidence gates, which directly contradicts FR-1.6. Any future design must either update FR-1.6 or find an alternative evidence verification mechanism within the allowed tool set.

5. **Anvil's Step 5d Operational Readiness (observability, degradation, secrets) is a distinct verification tier:** This is separate from build/test/lint verification and catches production-readiness issues that CI/CD checks miss. Should be preserved when porting Anvil patterns to multi-agent pipelines.

6. **Anvil's session_store recall pattern provides per-file regression history:** This is distinct from VS Code `store_memory` (which stores broad patterns). Per-file query capability (`WHERE file_path LIKE '%{filename}%'`) for regressions, reverts, and failures is a unique Anvil capability that `store_memory` cannot replicate.

7. **`git add -A` must be explicitly assigned before `git diff --staged` review:** In a multi-agent pipeline, no agent will independently stage changes unless explicitly instructed. Without staging, `git diff --staged` returns empty, causing silent review bypass.

8. **Feature-level code review on full `git diff --staged` doesn't scale:** For 20+ task features with 50+ files, the full staged diff will exceed reviewer context windows. Per-wave or per-risk-level review partitioning is needed for scalability.

## Citations

- Anvil Step 5c, 5d: `Anvil/anvil.agent.md` lines 215-258
- Anvil session_store: `Anvil/anvil.agent.md` lines 120-130
- Anvil `anvil_checks` schema: `Anvil/anvil.agent.md` lines 62-75
- feature.md FR-1.6: `docs/feature/next-gen-multi-agent-system/feature.md` line 642
- feature.md FR-5.3: `docs/feature/next-gen-multi-agent-system/feature.md` line 675
- feature.md CR-4: `docs/feature/next-gen-multi-agent-system/feature.md` lines 537-553
- design.md orchestrator tools: `docs/feature/next-gen-multi-agent-system/design.md` lines 170-180
- design.md sequence flow stale read: `docs/feature/next-gen-multi-agent-system/design.md` ~line 1580
