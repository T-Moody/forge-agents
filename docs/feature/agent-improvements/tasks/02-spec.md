# Task 02: Write Spec Agent

## Task Goal

Create the complete `spec.agent.md` file in `NewAgentsAndPrompts/`, rewriting the spec agent from scratch following the canonical structure and incorporating all improvements specified in the design.

**Output file:** `NewAgentsAndPrompts/spec.agent.md`

## depends_on

none

## In-Scope

- Complete rewrite of `spec.agent.md` following the canonical agent file structure
- Frontmatter with `name: spec` and updated description (add "testable")
- Role preamble (3 sentences)
- `detailed thinking on` directive with experimental comment
- **Inputs** section (initial-request.md, analysis.md)
- **Outputs** section (feature.md)
- **Operating Rules** block (shared block — spec variant, Rule 5: semantic_search, grep_search, read_file)
- **Workflow** (updated: adds structured edge cases guidance + self-verification step)
- **feature.md Contents** (updated: structured edge case format with Input/Condition, Expected Behavior, Severity if Missed)
- **Completion Contract** (DONE/ERROR only)
- **Anti-Drift Anchor** (exact text from design.md)

## Out-of-Scope

- Modifying files in `.github/`
- Adding NEEDS_REVISION (spec uses only DONE/ERROR)
- Changing the fundamental content of feature.md beyond adding structured edge cases

## Acceptance Criteria

1. File exists at `NewAgentsAndPrompts/spec.agent.md` and is non-empty
2. Frontmatter has `name: spec` and description including "testable"
3. Role preamble follows standard pattern with NEVER constraints
4. Contains `## Operating Rules` with spec-specific Rule 5
5. Workflow includes self-verification step (verify acceptance criteria are testable, requirements have criteria, edge cases cover failure modes, no contradictions)
6. feature.md Contents specifies structured edge case format: Input/Condition, Expected Behavior, Severity if Missed
7. Completion contract is two-state (DONE/ERROR only)
8. Anti-drift anchor is last section with exact text: "You are the **Spec Agent**. You write formal requirements specifications. You never write code, designs, or plans. You never implement anything. Stay as spec."
9. Sections in canonical order

## Test Requirements

1. **Structure test:** Verify canonical sections exist in correct order
2. **Self-verification test:** Verify workflow step 5 includes the 4 verification checks from design.md
3. **Edge case format test:** Verify feature.md Contents includes Input/Condition, Expected Behavior, Severity if Missed
4. **Cross-agent consistency:** Verify completion contract format matches orchestrator expectations

## Implementation Steps

1. Read design.md § Per-Agent Design → 3. `spec.agent.md` for target state and content specs
2. Read design.md § Cross-Cutting Patterns for shared blocks (Operating Rules, Anti-Drift, Completion Contract, thinking directive)
3. Read design.md § Agent File Structure Standard for canonical order
4. Create `NewAgentsAndPrompts/spec.agent.md` with:
   - YAML frontmatter
   - `# Spec Agent Workflow` title
   - Role preamble
   - `detailed thinking on` directive
   - `## Inputs` (initial-request.md, analysis.md)
   - `## Outputs` (feature.md)
   - `## Operating Rules` (spec variant)
   - `## Workflow` (5-step: read initial-request, review analysis, research gaps, define requirements, self-verify)
   - `## feature.md Contents` (updated with structured edge case format)
   - `## Completion Contract` (DONE/ERROR)
   - `## Anti-Drift Anchor`

## Estimated Effort

Low

## Completion Checklist

- [x] File created at `NewAgentsAndPrompts/spec.agent.md`
- [x] Canonical section order followed
- [x] Operating Rules block with spec-specific Rule 5
- [x] Self-verification step in workflow
- [x] Structured edge case format in feature.md Contents
- [x] Completion contract is two-state only
- [x] Anti-drift anchor is last section with exact text
- [x] No hardcoded references
- [x] `detailed thinking on` directive with experimental comment
