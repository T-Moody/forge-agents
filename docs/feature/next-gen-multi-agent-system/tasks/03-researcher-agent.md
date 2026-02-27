# Task 03: Researcher Agent Definition

## Task Goal

Create `NewAgents/.github/agents/researcher.agent.md` — the codebase investigation agent (1 definition, dispatched as 4 parallel instances with distinct focus areas).

## depends_on

01, 02

## agent

implementer

## In-Scope

- Agent definition following the template from design.md §Decision 10
- Role: focused codebase investigation producing typed YAML output + human-readable Markdown companion
- 4 focus areas: architecture, impact, dependencies, patterns
- Input schema reference: `initial-request.md`, codebase
- Output schema reference: `research-output` schema from schemas.md
- Output paths: `research/<focus>.yaml` + `research/<focus>.md`
- Tool access: read-only tools (`read_file`, `list_dir`, `grep_search`, `semantic_search`, `file_search`)
- Tool restriction: MUST NOT modify any files except writing its own outputs
- Completion contract: DONE | ERROR
- Self-verification: verify output matches research-output schema, verify findings are cited
- Anti-drift anchor

## Out-of-Scope

- Dispatch logic (handled by orchestrator)
- Schema definitions (in schemas.md)
- How 4 instances are coordinated (orchestrator responsibility)

## Acceptance Criteria

1. File exists at `NewAgents/.github/agents/researcher.agent.md`
2. Follows agent definition template: header, role, input/output schema, workflow, contract, rules, self-verification, tools, anti-drift
3. References `research-output` schema from schemas.md
4. Lists exactly 5 allowed tools: `read_file`, `list_dir`, `grep_search`, `semantic_search`, `file_search`
5. Completion contract specifies only DONE | ERROR (no NEEDS_REVISION)
6. Workflow describes: receive focus area → investigate codebase → structure findings → produce typed YAML + companion Markdown
7. Outputs defined as `research/<focus>.yaml` and `research/<focus>.md`
8. Self-verification section checks schema compliance and citation presence
9. Anti-drift anchor present

## Estimated Effort

Medium

## Test Requirements

- Grep for completion contract section with DONE | ERROR
- Grep for self-verification section
- Grep for anti-drift anchor
- Verify tool list matches design.md §Decision 2 (Researcher)

## Implementation Steps

1. Read design.md §Decision 2, Researcher agent detail for inputs, outputs, tools, contract
2. Read design.md §Decision 10 for agent definition template structure
3. Reference existing `.github/agents/researcher.agent.md` for research focus partitioning patterns
4. Create `NewAgents/.github/agents/researcher.agent.md` with:
   - Header block (type, pipeline step 1, inputs, outputs)
   - Role & Purpose (focused codebase investigation)
   - Input Schema (initial-request.md, codebase access)
   - Output Schema (reference schemas.md#research-output)
   - Workflow (receive focus → investigate → structure → output)
   - Completion Contract (DONE | ERROR)
   - Operating Rules (read-only, structured findings, cite sources)
   - Self-Verification (schema check, citation check)
   - Tool Access (5 read-only tools)
   - Anti-Drift Anchor
5. Self-verify all template sections present

## Relevant Context from design.md

- §Decision 2 (lines ~145–155) — Researcher: inputs, outputs, tools, restrictions, contract
- §Decision 3, Step 1 (lines ~315–325) — Research dispatch: ×4 parallel, gate ≥2/4 DONE
- §Decision 10 (lines ~1200–1260) — Agent definition template structure

## Completion Checklist

- [x] File created at correct path
- [x] All template sections present (Header, Role & Purpose, Input Schema, Output Schema, Workflow, Completion Contract, Operating Rules, Self-Verification, Tool Access, Research Focus Areas, Anti-Drift Anchor)
- [x] Schema reference to schemas.md present (lines 39, 45, 148)
- [x] Tool list correct (5 read-only tools: read_file, list_dir, grep_search, semantic_search, file_search)
- [x] Completion contract: DONE | ERROR only (NEEDS_REVISION explicitly excluded)
- [x] Anti-drift anchor present

TDD skipped: documentation-only task with no behavioral code; no test framework applicable.
