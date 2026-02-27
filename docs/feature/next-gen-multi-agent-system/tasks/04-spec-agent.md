# Task 04: Spec Agent Definition

## Task Goal

Create `NewAgents/.github/agents/spec.agent.md` — the feature specification agent with pushback system.

## depends_on

01, 02

## agent

implementer

## In-Scope

- Agent definition following design.md §Decision 10 template
- Role: produce feature specification from research outputs + initial request
- Pushback system: evaluate request for concerns, surface with structured multiple-choice via `ask_questions` (interactive: user decides to proceed/modify/abandon; autonomous: log and proceed)
- Input: all research outputs (typed YAML), `initial-request.md`
- Output: `spec-output.yaml` (typed spec-output schema) + `feature.md` (human-readable companion)
- Tool access: `read_file`, `list_dir`, `grep_search`, `semantic_search`, `file_search`, `create_file`, `replace_string_in_file`, `ask_questions` (pushback in interactive mode)
- Completion contract: DONE | ERROR
- Structured requirements format with acceptance criteria
- Self-verification + anti-drift anchor

## Out-of-Scope

- Schema definition (in schemas.md)
- Pushback evaluation at Step 0 (that's lightweight orchestrator-level, not Spec's pushback)
- Approval gate management (orchestrator responsibility)

## Acceptance Criteria

1. File exists at `NewAgents/.github/agents/spec.agent.md`
2. Follows agent definition template with all required sections
3. References `spec-output` schema from schemas.md
4. Pushback system documented: how concerns are identified, structured multiple-choice format via `ask_questions`, interactive vs autonomous behavior
5. Tool list includes `ask_questions` for interactive pushback
6. Completion contract: DONE | ERROR
7. Workflow includes: read research → evaluate pushback → produce requirements → structure output → self-verify
8. Output schema references match schemas.md
9. Self-verification and anti-drift anchor present

## Estimated Effort

Medium

## Test Requirements

- Grep for pushback-related sections
- Grep for completion contract DONE | ERROR
- Grep for `ask_questions` in tool access
- Verify self-verification and anti-drift sections present

## Implementation Steps

1. Read design.md §Decision 2, Spec agent detail for inputs, outputs, tools, contract
2. Read design.md §Decision 3, Step 2 for spec dispatch flow and pushback evaluation
3. Read design.md §Decision 9 for pushback behavior in interactive vs autonomous modes
4. Read feature.md §Functional Requirements FR-3 for pushback system requirements
5. Create `NewAgents/.github/agents/spec.agent.md` with full template structure
6. Self-verify all sections present

## Relevant Context from design.md

- §Decision 2 (lines ~155–165) — Spec: inputs, outputs, tools, contract
- §Decision 3, Step 2 (lines ~325–335) — Spec dispatch, pushback evaluation
- §Decision 9 (lines ~1050–1110) — Approval mode, pushback behavior

## Completion Checklist

- [x] File created at correct path
- [x] All template sections present
- [x] Pushback system documented
- [x] Schema reference to schemas.md present
- [x] Tool list correct (includes ask_questions)
- [x] Anti-drift anchor present

## Implementation Notes

- TDD skipped: agent definition is a Markdown file (no behavioral code, no test framework applicable)
- Verification performed via grep-based checks confirming all acceptance criteria sections are present
- Pushback system documented with full interactive/autonomous mode behavior, structured multiple-choice format, concern severity classification, and response handling
- Schema references point to schemas.md §Schema 2 (research-output inputs) and §Schema 3 (spec-output)
