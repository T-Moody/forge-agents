# Task 11: Write Documentation-Writer Agent

## Task Goal

Create the new `documentation-writer.agent.md` file in `NewAgentsAndPrompts/`. This is a **new agent** that does not exist in the current Forge framework. It handles documentation tasks when assigned via per-task agent routing.

**Output file:** `NewAgentsAndPrompts/documentation-writer.agent.md`

## depends_on

07

## Cluster

**Cluster B (Agent Routing) — Consumer.** This agent is invoked only via the `agent: documentation-writer` field in task files. Depends on:

- Planner (Task 07) defining `documentation-writer` as a valid agent value
- Orchestrator (Task 09) reading the `agent` field and dispatching to this agent

## In-Scope

- **Create new agent file** from scratch following canonical structure
- Frontmatter with `name: documentation-writer` and description
- Role preamble (3 sentences — generates documentation, read-only access to source, never modifies code)
- `detailed thinking on` directive
- **Inputs (STRICT)** section (task file, feature.md, design.md, entire codebase read-only)
- **Outputs** section (documentation files as specified in task, updated task file)
- **Operating Rules** block (documentation-writer variant, Rule 5: semantic_search, grep_search, read_file, never modify source)
- **Read-Only Enforcement** (cannot modify source code, tests, or config — only writes documentation files)
- **Capabilities** section (NEW — API docs, architectural diagrams, README updates, code-doc parity, coverage matrix)
- **Workflow** (6-step: read task → read context → analyze source → generate docs → verify accuracy delta-only → update task)
- **Completion Contract** (DONE/ERROR only — does NOT use NEEDS_REVISION)
- **Anti-Drift Anchor**

## Out-of-Scope

- Modifying files in `.github/`
- Adding this agent to the core pipeline (it is invoked only via per-task routing)
- Modifying source code, tests, or configuration
- Adding NEEDS_REVISION (uses only DONE/ERROR)

## Acceptance Criteria

1. File exists at `NewAgentsAndPrompts/documentation-writer.agent.md` and is non-empty (TS-12)
2. Defines role, workflow, completion contract, and anti-drift anchor (TS-12)
3. Read-Only Enforcement: explicitly states "MUST NOT modify source code, test files, or configuration files" (TS-12)
4. Agent is NOT referenced in core 8-step pipeline — only invocable via per-task routing (TS-12)
5. Capabilities include: API Documentation (OpenAPI/Swagger), Architectural Diagrams (Mermaid), README Updates, Code-Documentation Parity, Documentation Coverage Matrix
6. Workflow step 5 uses delta-only parity verification (get_changed_files or equivalent)
7. Completion contract: DONE: `<task-id>` / ERROR: `<reason>`
8. Anti-drift anchor: "You are the **Documentation Writer**. You generate and maintain documentation. You never modify source code or tests. You write documentation files only. Stay as documentation writer."
9. Frontmatter `name` matches filename stem: `documentation-writer`

## Test Requirements

1. **Existence test (TS-12):** File exists with role, workflow, contract, anchor
2. **Read-only test (TS-12):** File states source code is read-only
3. **Not in pipeline test (TS-12):** File is self-contained; not referenced as a core pipeline step
4. **Capabilities test:** Verify 5 documentation capabilities are listed
5. **Delta-only test:** Verify parity check is against changed files only

## Implementation Steps

1. Read design.md § Per-Agent Design → 11. `documentation-writer.agent.md` for complete content specifications
2. Read design.md § Cross-Cutting Patterns for shared blocks (completion contract = DONE/ERROR only, tool preferences for doc-writer)
3. Create `NewAgentsAndPrompts/documentation-writer.agent.md` with:
   - YAML frontmatter (`name: documentation-writer`, description)
   - `# Documentation Writer Agent Workflow` title
   - Role preamble (from design.md)
   - `detailed thinking on` directive
   - `## Inputs (STRICT)` (task file, feature.md, design.md, codebase read-only)
   - `## Outputs` (documentation files, updated task file)
   - `## Operating Rules` (documentation-writer variant)
   - `## Read-Only Enforcement` (from design.md)
   - `## Capabilities` (5 capabilities from design.md)
   - `## Workflow` (6-step from design.md with delta-only verification)
   - `## Completion Contract` (DONE/ERROR)
   - `## Anti-Drift Anchor`

## Estimated Effort

Low

## Completion Checklist

- [x] File created at `NewAgentsAndPrompts/documentation-writer.agent.md`
- [x] Complete agent definition (not a stub)
- [x] Read-Only Enforcement section present
- [x] 5 Capabilities listed (API docs, diagrams, README, parity, coverage)
- [x] Workflow includes delta-only parity verification
- [x] Completion contract is two-state (DONE/ERROR)
- [x] Anti-drift anchor with exact text
- [x] Not part of core pipeline (invoked only via agent routing)
- [x] Frontmatter name matches filename stem
- [x] Operating Rules with doc-writer specific Rule 5
