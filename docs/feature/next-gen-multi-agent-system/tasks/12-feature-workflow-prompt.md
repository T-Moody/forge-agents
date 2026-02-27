# Task 12: Feature Workflow Prompt

## Task Goal

Create `NewAgents/.github/prompts/feature-workflow.prompt.md` — the entry point prompt file that binds user feature requests to the orchestrator agent.

## depends_on

01

## agent

implementer

## In-Scope

- Prompt file with YAML frontmatter (`mode: agent`, `agent: orchestrator`, variables)
- Variables:
  - `USER_FEATURE`: the feature request from the user
  - `APPROVAL_MODE`: autonomous | interactive (default: autonomous)
- Key artifacts reference table pointing to agent definitions, schema reference, dispatch patterns
- Pipeline execution rules binding to orchestrator.agent.md
- Format matching VS Code prompt file conventions

## Out-of-Scope

- Orchestrator agent definition (Task 11)
- Schema definitions (Task 01)
- All other agent definitions

## Acceptance Criteria

1. File exists at `NewAgents/.github/prompts/feature-workflow.prompt.md`
2. YAML frontmatter includes `mode: agent` and `agent: orchestrator`
3. Variables defined: `USER_FEATURE` (required), `APPROVAL_MODE` (has default: autonomous)
4. Key artifacts reference table points to correct paths under `NewAgents/.github/`
5. Pipeline execution rules section references orchestrator.agent.md
6. Template variables use `{{VARIABLE_NAME}}` syntax
7. File is concise and focused (entry point, not documentation)

## Estimated Effort

Low

## Test Requirements

- Verify YAML frontmatter parses correctly (mode, agent, variables)
- Verify all referenced paths exist (after all other tasks complete)
- Verify variable syntax is correct

## Implementation Steps

1. Read design.md §Decision 10, Prompt File Structure for exact format specification
2. Reference existing `.github/prompts/feature-workflow.prompt.md` for formatting conventions
3. Create `NewAgents/.github/prompts/feature-workflow.prompt.md` with:
   - YAML frontmatter (mode, agent, variables)
   - Title section
   - Initial Request section with `{{USER_FEATURE}}` variable
   - Execution Mode section with `{{APPROVAL_MODE}}` variable
   - Key Artifacts Reference table
   - Pipeline Execution Rules (4 rules from design.md)
4. Self-verify frontmatter format

## Relevant Context from design.md

- §Decision 10 (lines ~1260–1290) — Prompt File Structure with exact format and rules

## Completion Checklist

- [x] File created at correct path
- [x] YAML frontmatter correct
- [x] Variables defined with descriptions and defaults
- [x] Artifacts reference table present
- [x] Pipeline execution rules present

## Notes

- TDD skipped: configuration-only task (markdown prompt file, no behavioral code)
- Design.md §Decision 10 specified `mode: agent` and `variables:` in YAML frontmatter, but VS Code prompt files don't support these attributes. Adapted to use `name`, `agent`, and `description` (supported attributes). Template variables (`{{USER_FEATURE}}`, `{{APPROVAL_MODE}}`) remain in the body as designed.
