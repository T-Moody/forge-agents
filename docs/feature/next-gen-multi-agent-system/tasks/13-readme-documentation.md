# Task 13: README Documentation

## Task Goal

Create `NewAgents/README.md` — the pipeline overview document with Mermaid diagram, agent inventory table, and quick-start usage instructions.

## depends_on

11, 12

## agent

documentation-writer

## In-Scope

- Pipeline overview: one-paragraph summary of the system
- Mermaid flowchart diagram matching design.md §Decision 10 (Steps 0–9 with all gates, loops, and decisions)
- Agent inventory table: 9 agents with roles, pipeline step, input/output summary
- Reference documents table: schemas.md, dispatch-patterns.md, severity-taxonomy.md
- Quick-start usage: how to invoke the pipeline via feature-workflow.prompt.md
- Architecture highlights: key design decisions, what makes this different from Forge/Anvil
- File structure overview of `NewAgents/` directory

## Out-of-Scope

- Agent definitions (Tasks 03–11)
- Schema content (Task 01)
- Detailed design rationale (in design.md)

## Acceptance Criteria

1. File exists at `NewAgents/README.md`
2. Contains Mermaid pipeline diagram matching design.md §Decision 10 (Steps 0–9, both loops, both review phases, gates)
3. Agent inventory table lists all 9 agents with: name, role, pipeline step, completion contract states
4. Reference documents section lists 3 shared references
5. Quick-start section explains how to use feature-workflow.prompt.md
6. Directory structure overview present
7. Diagram renders correctly in GitHub/VS Code Markdown preview

## Estimated Effort

Low

## Test Requirements

- Verify Mermaid diagram contains all pipeline steps (S0–S9 + S3b + S7 + S8b)
- Verify agent count in table = 9
- Verify all file paths referenced are accurate

## Implementation Steps

1. Read design.md §Decision 10, README with Mermaid Diagram for the exact Mermaid flowchart definition
2. Read design.md §Decision 2 for agent inventory summary (9 agents with roles)
3. Read design.md §Decision 10, Output Directory Layout for file structure
4. Create `NewAgents/README.md` with:
   - Title and one-paragraph overview
   - Mermaid pipeline diagram (copy and verify from design.md §Decision 10)
   - Agent inventory table
   - Reference documents table
   - Quick-start usage section
   - Directory structure overview
5. Verify Mermaid syntax is valid

## Relevant Context from design.md

- §Decision 10, README with Mermaid Diagram (lines ~1300–1370) — Mermaid flowchart definition
- §Decision 2 (lines ~100–120) — Agent inventory table (9 agents)
- §Decision 10, Output Directory Layout (lines ~1170–1200) — File structure

## Completion Checklist

- [x] File created at correct path
- [x] Mermaid diagram present with all pipeline steps
- [x] Agent inventory table with 9 agents
- [x] Reference documents listed
- [x] Quick-start usage section present
- [x] Directory structure overview present

## Implementation Status

**DONE** — `NewAgents/README.md` created by documentation-writer agent (Task 13).
