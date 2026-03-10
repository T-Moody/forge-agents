# Global Rules

Cross-cutting rules for all agents. Read this document in full — no section references needed.

## Completion Contract

Every agent output MUST end with a typed completion block:

```yaml
completion:
  status: "DONE" | "NEEDS_REVISION" | "ERROR"
  summary: "≤200 characters describing the outcome"
  output_paths:
    - "relative/path/to/output.yaml"
```

**Required fields:** `status`, `summary`, `output_paths` — all three are mandatory.

**Validation rule:** Any status value other than DONE, NEEDS_REVISION, or ERROR is treated as ERROR by the orchestrator. This prevents undefined behavior from malformed outputs.

## Retry Policy

- **Transient errors** (network timeout, tool unavailable): retry once, then fail.
- **Deterministic errors** (file not found, permission denied, invalid input): fail immediately — do not retry.

## Anti-Hallucination

Never claim verification passed without executing actual checks. If a tool is unavailable or a check was not run, report it — do not fabricate results. Evidence must exist before it is referenced.

## Feedback Loop Limits

All feedback loops have hard iteration caps to prevent infinite cycling:

| Loop                   | Path                                       | Max Iterations |
| ---------------------- | ------------------------------------------ | -------------- |
| Implementation-Testing | Implementer → Tester → Implementer         | 3              |
| Code Review Fix        | Reviewer → Implementer → Tester → Reviewer | 2              |
| Design Review          | Reviewer → Architect → Reviewer            | 1              |
| Plan Refinement        | Planner → Reviewer → Planner               | 1              |

After the limit is reached, the orchestrator escalates (ERROR status) rather than retrying.

Counter reset: the Implementation-Testing counter resets to 0 at the start of each Code Review round.

## Output Path Convention

All agent outputs MUST be written to:

```
docs/feature/<feature-slug>/
```

Subdirectories follow the agent's role (e.g., `implementation-reports/`, `research/`, `review-findings/`). Agents MUST NOT write files outside this directory tree.

## Tool Prohibitions

**VS Code `runTests` tool:** ALL agents MUST NOT use the VS Code `runTests` tool. It freezes agent execution in subagent contexts. Always use `run_in_terminal` with the appropriate CLI test command (e.g., `dotnet test`, `npm test`, `pytest`).

**No file redirect:** NEVER redirect terminal output to files (`>`, `>>`, `| tee`, `Out-File`, `Set-Content`). Read output directly from the terminal response. This prevents stale files and cross-agent contamination.

## Git Safety

Only the orchestrator may run git staging (add) and commit commands. All other agents MUST NOT run `git add`, `git commit`, `git push`, `git reset`, or any write-mode git operations.

Permitted for non-orchestrator agents (read-only): `git diff`, `git status`.

## Risk Classification

Risk level determines pipeline scaling — more risk means more scrutiny:

| Level | Label  | Research  | Design Review | Reviewers | Dispatch Estimate |
| ----- | ------ | --------- | ------------- | --------- | ----------------- |
| 🟢    | Green  | 2 queries | None          | 2         | ~8-10 dispatches  |
| 🟡    | Yellow | 3 queries | None          | 2         | ~12-15 dispatches |
| 🔴    | Red    | 4 queries | 1 round (max) | 3         | ~20-25 dispatches |

**Scaling behavior:**

- **🟢 Green:** Low-risk changes (docs, config, small features). Minimal research, no design review.
- **🟡 Yellow:** Moderate-risk changes (new features, refactors). Additional research depth. E2E testing when skills/contract present.
- **🔴 Red:** High-risk changes (architecture, security, breaking changes). Full research, embedded design review with 1 revision round, 3 independent reviewers.

Risk level is assigned by the orchestrator at Setup (Step 1) based on the feature request scope and impact.
