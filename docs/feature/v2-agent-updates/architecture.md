# V2 Agent System Updates — Architecture Summary

## Feature Overview

This feature applies 9 targeted updates to the v2 multi-agent system (8 pipeline agents + global-rules.md + 2 prompts). The changes enforce tool restrictions, mandate web research behavior, improve orchestrator delegation discipline, reduce implementer build overhead, and fix known issues.

**Risk Level:** 🟡 (multi-file, cross-cutting but non-architectural)
**Direction:** B — Cross-cutting rules in global-rules.md + surgical per-agent edits
**Files Modified:** 10

---

## Key Decisions

### D-1: vscodeAskQuestions tool name (Medium confidence)

The orchestrator currently references `ask_questions` — a name not found in the official VS Code built-in tools documentation. Research and runtime observation indicate `vscodeAskQuestions` is the correct camelCase name. All references updated accordingly. If the name is wrong in a `tools:` array, VS Code silently ignores it (the tool becomes unavailable).

**Action:** Implementer should verify the tool name at implementation time by checking runtime tool availability.

### D-2: Individual tool names over tool sets

VS Code supports tool sets (`edit`, `search`, `runCommands`) but they're too coarse-grained for the v2 trust model. Individual camelCase tool names provide fine-grained control — e.g., restricting the orchestrator to `createFile` without `editFiles`.

### D-3: Cross-cutting rules in global-rules.md

Four new sections added to global-rules.md:

1. **Web Research Mandate** — When `web_research_enabled=true`, agents with `fetch` MUST use it
2. **User Interaction** — All user input via `vscodeAskQuestions` with multiple-choice; never free-form
3. **E2E Artifact Storage** — Screenshots/traces/logs in `docs/feature/<slug>/e2e-artifacts/`
4. **Git Safety (expanded)** — No agent may create, checkout, or delete branches

### D-4: Quick-fix always dispatches planner

Orchestrator's inline plan generation for quick-fix mode violates subagent-only (FR-3). Removed. The quick-fix pipeline now dispatches the planner in quick-fix mode. One extra dispatch (~seconds) for full compliance.

### D-5: Batch builds in implementer GREEN phase

Changed from "run problems after every file edit" to "after all production code edits, run problems once and test once." Reduces worst-case build count from N+6 to ~4 builds.

### D-6: Orchestrator tools: restriction

```yaml
tools:
  - vscodeAskQuestions
  - runSubagent
  - runInTerminal
  - getTerminalOutput
  - listDirectory
  - readFile
  - createFile
  - createDirectory
```

Excludes: `textSearch`, `codebase`, `fileSearch`, `editFiles`, `problems`, `changes`, `fetch`. Platform-level enforcement of subagent-only.

### D-10: Completion-contract-only reads

Orchestrator reads ONLY completion contracts from subagent outputs — `status`, `summary`, `output_paths`, plus routing fields (`clarifications_needed`, `plan_summary`, `verdict`). Full artifact analysis delegated to subagents. Prevents context window bloat.

---

## Trust Model Enforcement via tools: Arrays

| Agent        | Trust Tier | Tools                                                                                                                       |
| ------------ | ---------- | --------------------------------------------------------------------------------------------------------------------------- |
| orchestrator | T1         | vscodeAskQuestions, runSubagent, runInTerminal, getTerminalOutput, listDirectory, readFile, createFile, createDirectory     |
| researcher   | T3         | codebase, textSearch, fileSearch, listDirectory, readFile, createFile, fetch                                                |
| architect    | T3         | codebase, textSearch, fileSearch, listDirectory, readFile, createFile, fetch                                                |
| planner      | —          | readFile, listDirectory, createFile, fileSearch, textSearch                                                                 |
| implementer  | T2         | readFile, editFiles, createFile, listDirectory, textSearch, fileSearch, runInTerminal, getTerminalOutput, problems, changes |
| tester       | T2         | readFile, createFile, listDirectory, runInTerminal, getTerminalOutput, problems, textSearch, fileSearch, openSimpleBrowser  |
| reviewer     | T1         | readFile, listDirectory, textSearch, fileSearch, createFile, changes, problems, codebase                                    |
| knowledge    | T1         | readFile, listDirectory, createFile, textSearch, fileSearch                                                                 |

**Key restrictions enforced:**

- Orchestrator: no `editFiles`, `textSearch`, `codebase`, `fileSearch` (delegates analysis)
- Researcher/Architect: no `editFiles`, `runInTerminal` (read-only + gated fetch)
- Reviewer: no `editFiles`, `runInTerminal` (read-only analysis)
- Knowledge: no `editFiles`, `runInTerminal` (read-only + create only)
- Implementer: no `fetch`, `runSubagent` (no web access, no delegation)

---

## File Change Summary

| #   | File                                      | Changes                                                                                                                    |
| --- | ----------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| 1   | `v2/.github/agents/global-rules.md`       | Add YAML frontmatter + 4 new cross-cutting sections                                                                        |
| 2   | `v2/.github/agents/orchestrator.agent.md` | Add tools:, ask_questions→vscodeAskQuestions, remove branch creation, remove quick-fix inline plan, add context management |
| 3   | `v2/.github/agents/researcher.agent.md`   | Add tools:, mandatory web research language                                                                                |
| 4   | `v2/.github/agents/architect.agent.md`    | Add tools:, mandatory web research language                                                                                |
| 5   | `v2/.github/agents/implementer.agent.md`  | Add tools:, batch builds in GREEN phase                                                                                    |
| 6   | `v2/.github/agents/tester.agent.md`       | Add tools:, E2E artifact storage path                                                                                      |
| 7   | `v2/.github/agents/reviewer.agent.md`     | Add tools:                                                                                                                 |
| 8   | `v2/.github/agents/planner.agent.md`      | Add tools:                                                                                                                 |
| 9   | `v2/.github/agents/knowledge.agent.md`    | Add tools:                                                                                                                 |
| 10  | `v2/.github/prompts/quick-fix.prompt.md`  | Add Planning step (planner always dispatched)                                                                              |

---

## Risks and Mitigations

| Risk                                                       | Impact                                                                        | Mitigation                                                                     |
| ---------------------------------------------------------- | ----------------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `vscodeAskQuestions` may not be the correct YAML tool name | Orchestrator loses interactive capability if tools: array includes wrong name | VS Code silently ignores unknown names; implementer verifies at runtime        |
| Orchestrator tools: restriction may be too tight           | Pipeline steps requiring currently-unnamed tools fail                         | readFile covers completion contract reads; createFile covers pipeline-log.yaml |
| Quick-fix pipeline adds one planner dispatch               | Slightly slower quick-fix execution                                           | Planner produces minimal output; ~seconds overhead                             |
| global-rules.md line growth                                | Approaches shared doc budget                                                  | ~120 lines total — well within limits                                          |
