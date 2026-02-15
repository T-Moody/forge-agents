# Task 04: Write Critical-Thinker Agent

## Task Goal

Create the complete `critical-thinker.agent.md` file in `NewAgentsAndPrompts/`, performing a **full rewrite** that fixes two pre-existing bugs (missing completion contract, missing output specification) and restructures the agent from vague conversational prompts into a structured adversarial design review agent.

**Output file:** `NewAgentsAndPrompts/critical-thinker.agent.md`

## depends_on

none

## In-Scope

- **Full rewrite** of `critical-thinker.agent.md` — current file is ~30 lines of conversational prompts; target is a structured agent
- Frontmatter with `name: critical-thinker` and precise description ("adversarial design review")
- Role preamble (3 sentences — NO interactive question-asking, NO suggesting solutions, produces written review document)
- `detailed thinking on` directive with experimental comment
- **Inputs** section (initial-request.md, design.md, feature.md)
- **Outputs** section — **BUG FIX: explicitly specifies `design_critical_review.md`**
- **Operating Rules** block (critical-thinker variant, Rule 5: semantic_search, grep_search, read_file)
- **Workflow** (complete rewrite — 8-step structured risk analysis, NOT conversational Q&A)
- **Risk Categories** section (NEW — 6 structured categories: security, scalability, maintainability, backwards compat, edge cases, performance)
- **design_critical_review.md Contents** (NEW — defines output structure)
- **Completion Contract** — **BUG FIX: adds missing contract** — THREE-STATE: DONE/NEEDS_REVISION/ERROR (critical-thinker is one of only 3 agents that uses NEEDS_REVISION)
- **Anti-Drift Anchor**

## Out-of-Scope

- Interactive Q&A behavior (removed — agent runs non-interactively in pipeline)
- Solution proposals (agent identifies problems only)
- Modifying files in `.github/`

## Acceptance Criteria

1. File exists at `NewAgentsAndPrompts/critical-thinker.agent.md` and is non-empty
2. **BUG FIX verified:** Contains `## Completion Contract` section with `DONE:`, `NEEDS_REVISION:`, and `ERROR:` formats
3. **BUG FIX verified:** Contains `## Outputs` section specifying `design_critical_review.md`
4. Role preamble says "You NEVER propose solutions — you identify problems" and "You NEVER ask interactive questions — you produce a written review document"
5. Contains `## Risk Categories` with 6 categories: security vulnerabilities, scalability bottlenecks, maintainability concerns, backwards compatibility risks, edge cases not covered, performance implications
6. Each risk category requires: What, Where, Likelihood, Impact, Assumption at risk
7. design_critical_review.md Contents defines: title/summary, overall risk level, risks by category, requirement coverage gaps, key assumptions, recommendations
8. Workflow includes self-verification step: verify each risk is grounded in specific technical details
9. `NEEDS_REVISION` routing target is the designer (per design.md routing table)
10. Anti-drift anchor with exact text: "You are the **Critical Thinker**. You perform adversarial design reviews. You never write code, designs, plans, or specifications. You identify risks and weaknesses. Stay as critical thinker."

## Test Requirements

1. **Bug fix test (TS-2):** Search for `## Completion Contract` → must exist with DONE/NEEDS_REVISION/ERROR
2. **Bug fix test (TS-2):** Search for `design_critical_review.md` → must be specified as output
3. **Risk categories test:** Verify 6 categories with structured format (what, where, likelihood, impact, assumption)
4. **Non-interactive test:** File must NOT contain "ask questions", "encourage the engineer", "play devil's advocate" (old conversational patterns)
5. **Three-state test:** Completion contract includes NEEDS_REVISION with designer routing guidance

## Implementation Steps

1. Read design.md § Per-Agent Design → 5. `critical-thinker.agent.md` — this is marked as FULL REWRITE with complete content specifications
2. Read design.md § Cross-Cutting Patterns for:
   - Pattern 3: Completion Contract — use THREE-STATE format for critical-thinker (DONE/NEEDS_REVISION/ERROR)
   - Pattern 2: Anti-Drift Anchor (critical-thinker row)
   - Pattern 1: Operating Rules (critical-thinker Rule 5)
3. **DO NOT reference the current critical-thinker file** — the design specifies a complete rewrite, not an update
4. Create `NewAgentsAndPrompts/critical-thinker.agent.md` with:
   - YAML frontmatter (`name: critical-thinker`, description about adversarial design review)
   - `# Critical Thinker Agent Workflow` title
   - Role preamble (from design.md — structured, non-conversational)
   - `detailed thinking on` directive
   - `## Inputs` (initial-request.md, design.md, feature.md)
   - `## Outputs` (design_critical_review.md — **fixes bug**)
   - `## Operating Rules` (critical-thinker variant)
   - `## Workflow` (8-step structured analysis from design.md)
   - `## Risk Categories` (6 categories with structured per-risk format)
   - `## design_critical_review.md Contents` (output format)
   - `## Completion Contract` (THREE-STATE — **fixes bug**)
   - `## Anti-Drift Anchor`

## Estimated Effort

Medium

## Completion Checklist

- [x] File created at `NewAgentsAndPrompts/critical-thinker.agent.md`
- [x] **BUG FIX:** Completion contract section exists with three states
- [x] **BUG FIX:** Output file `design_critical_review.md` explicitly specified
- [x] Full rewrite — no conversational Q&A style remains
- [x] 6 risk categories present with structured format
- [x] design_critical_review.md Contents section defines output structure
- [x] NEEDS_REVISION routes to designer (mentioned in contract guidance)
- [x] Self-verification step in workflow
- [x] Anti-drift anchor with exact text
- [x] No references to interactive dialogue or question-asking
