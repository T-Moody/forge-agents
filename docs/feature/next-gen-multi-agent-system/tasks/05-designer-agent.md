# Task 05: Designer Agent Definition

## Task Goal

Create `NewAgents/.github/agents/designer.agent.md` — the technical design agent with justification scoring.

## depends_on

01, 02

## agent

implementer

## In-Scope

- Agent definition following design.md §Decision 10 template
- Role: produce technical design with decision justifications and scoring
- Input: `spec-output.yaml`, all research outputs (typed YAML), `initial-request.md`, adversarial review verdicts (if revision mode)
- Output: `design-output.yaml` (typed design-output schema with justification scores) + `design.md` (human-readable companion)
- Justification scoring: every decision uses decision-record format (id, title, context, alternatives with pros/cons, selected, rationale, confidence, risk)
- Confidence levels defined: High/Medium/Low with concrete definitions
- Tool access: `read_file`, `list_dir`, `grep_search`, `semantic_search`, `file_search`, `create_file`, `replace_string_in_file`
- Completion contract: DONE | ERROR
- Self-verification + anti-drift anchor

## Out-of-Scope

- Schema definition (in schemas.md)
- Adversarial review dispatch (orchestrator responsibility)
- Risk classification system details (planner agent's responsibility)

## Acceptance Criteria

1. File exists at `NewAgents/.github/agents/designer.agent.md`
2. Follows agent definition template with all required sections
3. References `design-output` schema from schemas.md
4. Justification scoring mechanism documented with decision-record format from design.md §Decision 8
5. Confidence level definitions included (High/Medium/Low)
6. Completion contract: DONE | ERROR
7. Workflow includes: read spec + research → evaluate directions → make decisions → score justifications → produce output
8. Self-verification and anti-drift anchor present

## Estimated Effort

Medium

## Test Requirements

- Grep for justification scoring or decision-record references
- Grep for confidence level definitions
- Verify completion contract and anti-drift anchor

## Implementation Steps

1. Read design.md §Decision 2, Designer agent detail for inputs, outputs, tools, contract
2. Read design.md §Decision 8 for justification scoring mechanism (decision-record schema)
3. Read design.md §Decision 3, Step 3 for designer dispatch flow
4. Create `NewAgents/.github/agents/designer.agent.md` with full template structure
5. Self-verify all sections present

## Relevant Context from design.md

- §Decision 2 (lines ~198–205) — Designer: inputs, outputs, tools, contract
- §Decision 3, Step 3 (lines ~335–340) — Designer dispatch
- §Decision 8 (lines ~980–1020) — Justification scoring, confidence definitions, decision-record schema

## Completion Checklist

- [x] File created at correct path
- [x] All template sections present
- [x] Justification scoring documented
- [x] Confidence levels defined
- [x] Schema reference to schemas.md present
- [x] Anti-drift anchor present

## Implementation Notes

- TDD skipped: markdown agent definition file, no test framework applicable. Validated via grep-based checks per Test Requirements.
- All 8 acceptance criteria verified via grep search against the created file.
- Agent definition follows design.md §Decision 10 template structure exactly.
- Decision-record format from §Decision 8 embedded with full field definitions.
- Confidence levels (High/Medium/Low) with concrete definitions from §Decision 8 table.
