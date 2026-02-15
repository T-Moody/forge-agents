# Task 03: Write Designer Agent

## Task Goal

Create the complete `designer.agent.md` file in `NewAgentsAndPrompts/`, rewriting the designer agent from scratch following the canonical structure and incorporating security considerations, failure & recovery sections, and self-verification.

**Output file:** `NewAgentsAndPrompts/designer.agent.md`

## depends_on

none

## In-Scope

- Complete rewrite of `designer.agent.md` following canonical structure
- Frontmatter with `name: designer` and updated description ("including security considerations and failure analysis")
- Role preamble (3 sentences)
- `detailed thinking on` directive with experimental comment
- **Inputs** section (initial-request.md, analysis.md, feature.md, design_critical_review.md if exists for revision cycle)
- **Outputs** section (design.md)
- **Operating Rules** block (designer variant, Rule 5: semantic_search, grep_search, read_file)
- **Workflow** (updated: 10-step, adds security considerations, failure/recovery analysis, self-verification step)
- **design.md Contents** (updated: adds Security Considerations + Failure & Recovery subsections)
- **Completion Contract** (DONE/ERROR only — designer does NOT use NEEDS_REVISION)
- **Anti-Drift Anchor**

## Out-of-Scope

- Modifying files in `.github/`
- Adding NEEDS_REVISION (designer uses only DONE/ERROR)
- Changing the fundamental design.md structure beyond adding new subsections

## Acceptance Criteria

1. File exists at `NewAgentsAndPrompts/designer.agent.md` and is non-empty
2. Frontmatter has `name: designer` and description mentioning security and failure analysis
3. Inputs include `design_critical_review.md (if exists — read during revision cycle)`
4. Workflow includes step for security considerations (auth, authorization, data protection, input validation)
5. Workflow includes step for failure modes and recovery strategies
6. Workflow includes self-verification step (design addresses all requirements, every AC has implementation path, security addressed, failure modes identified)
7. design.md Contents includes **Security Considerations** subsection (auth/authz, data protection, threat model, input validation)
8. design.md Contents includes **Failure & Recovery** subsection (failure modes, retry/fallback, graceful degradation)
9. Completion contract is two-state (DONE/ERROR only)
10. Anti-drift anchor with exact text: "You are the **Designer**. You create technical design documents. You never write code, tests, or plans. You never implement anything. Stay as designer."

## Test Requirements

1. **Structure test:** Canonical sections in correct order
2. **Security section test:** design.md Contents includes Security Considerations with 4 sub-items
3. **Failure section test:** design.md Contents includes Failure & Recovery with 3 sub-items
4. **Self-verification test:** Workflow includes verification step with 4 checks
5. **Input test:** design_critical_review.md listed as conditional input

## Implementation Steps

1. Read design.md § Per-Agent Design → 4. `designer.agent.md` for target state and content specs
2. Read design.md § Cross-Cutting Patterns for shared blocks
3. Create `NewAgentsAndPrompts/designer.agent.md` with:
   - YAML frontmatter
   - `# Designer Agent Workflow` title
   - Role preamble
   - `detailed thinking on` directive
   - `## Inputs` (including conditional design_critical_review.md)
   - `## Outputs` (design.md)
   - `## Operating Rules` (designer variant)
   - `## Workflow` (10-step with security, failure/recovery, self-verification)
   - `## design.md Contents` (updated with Security Considerations + Failure & Recovery)
   - `## Completion Contract` (DONE/ERROR)
   - `## Anti-Drift Anchor`

## Estimated Effort

Low

## Completion Checklist

- [x] File created at `NewAgentsAndPrompts/designer.agent.md`
- [x] Canonical section order followed
- [x] Security Considerations in design.md Contents (4 items)
- [x] Failure & Recovery in design.md Contents (3 items)
- [x] Self-verification step in workflow
- [x] design_critical_review.md in inputs as conditional
- [x] Completion contract is two-state only
- [x] Anti-drift anchor with exact text
- [x] No hardcoded references
