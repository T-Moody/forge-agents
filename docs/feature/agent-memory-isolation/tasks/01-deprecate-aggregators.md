# Task 01: Deprecate Aggregator Agents

## Task Goal

Replace the contents of `ct-aggregator.agent.md`, `v-aggregator.agent.md`, and `r-aggregator.agent.md` with deprecation notices indicating their decision logic has moved to the orchestrator.

## depends_on

none

## agent

implementer

## In-Scope

- Replace full contents of all 3 aggregator files with deprecation notices
- Each deprecation notice must state where the decision logic moved (orchestrator)
- Each notice must state that sub-agents now write isolated memory files

## Out-of-Scope

- Modifying any other agent files (aggregator reference cleanup happens in other tasks)
- Deleting the files (they are deprecated, not deleted)

## Acceptance Criteria

- AC-1: All 3 aggregator files contain only a deprecation notice — no active Inputs, Outputs, Operating Rules, Workflow, or Completion Contract sections
- Each deprecation notice matches the text specified in design.md Migration & Backwards Compatibility section:
  - ct-aggregator: "DEPRECATED: CT cluster decision logic has moved to the orchestrator. Individual CT sub-agents write isolated memory files. The orchestrator reads these memories and applies severity-based routing directly."
  - v-aggregator: "DEPRECATED: V cluster decision logic has moved to the orchestrator. Individual V sub-agents write isolated memory files. The orchestrator reads these memories and applies the V decision table directly."
  - r-aggregator: "DEPRECATED: R cluster decision logic has moved to the orchestrator. Individual R sub-agents write isolated memory files. The orchestrator reads these memories and applies R completion logic directly."

## Estimated Effort

Low

## Test Requirements

TDD skipped: markdown-only task, no behavioral code.

## Implementation Steps

1. Read `ct-aggregator.agent.md`, `v-aggregator.agent.md`, `r-aggregator.agent.md` to confirm current contents
2. Replace the entire contents of `ct-aggregator.agent.md` with YAML frontmatter (retaining `name: ct-aggregator`) + deprecation heading + deprecation notice text
3. Replace the entire contents of `v-aggregator.agent.md` with YAML frontmatter (retaining `name: v-aggregator`) + deprecation heading + deprecation notice text
4. Replace the entire contents of `r-aggregator.agent.md` with YAML frontmatter (retaining `name: r-aggregator`) + deprecation heading + deprecation notice text

## Status

Complete

TDD skipped: markdown-only task, no behavioral code.

## Completion Checklist

- [x] `ct-aggregator.agent.md` contains only YAML frontmatter + deprecation notice
- [x] `v-aggregator.agent.md` contains only YAML frontmatter + deprecation notice
- [x] `r-aggregator.agent.md` contains only YAML frontmatter + deprecation notice
- [x] No active workflow instructions remain in any of the 3 files
- [x] Each deprecation notice references the orchestrator as the new location for decision logic
- [x] Each notice mentions isolated memory files
