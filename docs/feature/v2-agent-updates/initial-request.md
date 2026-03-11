# V2 Agent System Updates

## Changes Requested

1. **Enforce web research when enabled** — When `web_research_enabled` is true, ALL agents with fetch access MUST perform web research. It is not optional.
2. **E2E artifacts in feature slug folder** — Screenshots, images, and other E2E artifacts must be stored in `docs/feature/<slug>/`, not project root.
3. **Orchestrator subagent-only** — The orchestrator MUST use the subagent tool for ALL work. It should NEVER analyze or create code on its own.
4. **Prevent orchestrator context overload** — Orchestrator must delegate file reading and analysis to subagents to keep its context window focused.
5. **Reduce implementer builds** — Implementers should not constantly build the project, slowing things down. Batch changes before building.
6. **No branch creation by agents** — Agents should never create git branches on their own.
7. **Enforce ask_questions tool** — Agents should NEVER stop a request to ask free-form questions. They must use the ask_questions tool with multiple-choice options.

## Scope

All files in `v2/.github/agents/` and `v2/.github/prompts/`.

## Research Done

Extensive web research completed on:

- VS Code Copilot custom agents documentation
- Built-in tools reference (complete list of camelCase tool names)
- Subagent execution model (synchronous, parallel capable, isolated context windows)
- Context engineering best practices
- Tool restrictions via YAML frontmatter
- `user-invocable` vs `disable-model-invocation` behavior
