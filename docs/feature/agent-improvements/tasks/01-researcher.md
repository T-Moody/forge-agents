# Task 01: Write Researcher Agent

## Task Goal

Create the complete `researcher.agent.md` file in `NewAgentsAndPrompts/`, rewriting the researcher agent from scratch following the canonical structure and incorporating all improvements specified in the design.

**Output file:** `NewAgentsAndPrompts/researcher.agent.md`

## depends_on

none

## In-Scope

- Complete rewrite of `researcher.agent.md` following the canonical agent file structure (design.md § Agent File Structure Standard)
- Frontmatter with `name: researcher` and updated description
- Role preamble (3 sentences: what you do, what you NEVER do)
- `detailed thinking on` directive with `<!-- experimental: model-dependent -->` comment
- **Inputs** section for both focused and synthesis modes
- **Outputs** section for both modes (was implicit, now explicit)
- **Operating Rules** block (shared block — researcher variant, Rule 5: semantic_search, grep_search, file_search, read_file)
- **Mode 1: Focused Research** section including:
  - Research Focus Areas table (same as current)
  - **Retrieval Strategy** subsection (NEW — semantic_search → grep_search → merge → read_file → file_search)
  - Focused Research Rules (updated with hybrid retrieval guidance + decisions.md read convention)
  - Partial Analysis File Contents (updated with confidence metadata: confidence_level, coverage_estimate, gaps)
- **Mode 2: Synthesis** section (same structure as current)
- **Completion Contract** for both modes (DONE/ERROR only — researcher does NOT use NEEDS_REVISION)
- **Anti-Drift Anchor** (exact text from design.md table)

## Out-of-Scope

- Modifying any files in `.github/`
- Adding NEEDS_REVISION to the completion contract (researcher uses only DONE/ERROR)
- Changing the focus area types (architecture, impact, dependencies)
- Adding new modes beyond focused and synthesis

## Acceptance Criteria

1. File exists at `NewAgentsAndPrompts/researcher.agent.md` and is non-empty
2. Frontmatter has `name: researcher` and technology-agnostic description
3. Role preamble follows the pattern: "You are the **Research Agent**..." with NEVER constraints
4. Contains `## Operating Rules` with 5 rules; Rule 5 matches the researcher row from the design.md tool preferences table
5. Contains `### Retrieval Strategy` subsection prescribing: semantic_search → grep_search → merge/deduplicate → read_file → file_search
6. Partial Analysis File Contents includes confidence_level, coverage_estimate, and gaps fields
7. Focused Research Rules includes: "If `decisions.md` exists... read it and incorporate prior architectural decisions"
8. Completion contract specifies `DONE:` and `ERROR:` (two-state only)
9. Anti-drift anchor is the last section, contains exact text: "You are the **Researcher**. You investigate and document findings. You never modify source code, tests, or project files. You never make design decisions. Stay as researcher."
10. Sections appear in canonical order: Title → Preamble → thinking directive → Inputs → Outputs → Operating Rules → Workflow sections → Completion Contract → Anti-Drift Anchor

## Test Requirements

Content verification (not code tests — this is a markdown file):

1. **Structure test:** Verify all canonical sections exist in the correct order
2. **Retrieval Strategy test:** Verify the 5-step methodology is present with correct tool names
3. **Confidence metadata test:** Verify confidence_level, coverage_estimate, gaps are in the partial analysis format
4. **Decision log test:** Verify decisions.md read convention is in Focused Research Rules
5. **Cross-agent consistency test:** Verify completion contract format matches what the orchestrator expects (DONE: focus-area — summary / ERROR: reason)
6. **Anti-drift test:** Verify anchor is the LAST section with exact text from design.md

## Implementation Steps

1. Read design.md § Per-Agent Design → 2. `researcher.agent.md` for the complete target state outline and content specifications
2. Read design.md § Cross-Cutting Patterns for:
   - Pattern 1: Operating Rules Block (use researcher variant for Rule 5)
   - Pattern 2: Anti-Drift Anchor (use researcher row)
   - Pattern 3: Completion Contract Format (researcher uses only DONE/ERROR)
   - Pattern 4: `detailed thinking on` directive
3. Read design.md § Agent File Structure Standard for canonical section order
4. Create `NewAgentsAndPrompts/researcher.agent.md` with:
   - YAML frontmatter (`name: researcher`, updated `description`)
   - `# Research Agent Workflow` title
   - Role preamble paragraph (3 sentences from design.md § Role Preamble Pattern)
   - `detailed thinking on` directive with experimental comment
   - `## Inputs` with Focused Mode and Synthesis Mode subsections
   - `## Outputs` with Focused Mode and Synthesis Mode subsections
   - `## Operating Rules` (shared block, researcher Rule 5)
   - `## Mode 1: Focused Research` with:
     - `### Research Focus Areas` table (carry from current)
     - `### Retrieval Strategy` (NEW content from design.md)
     - `### Focused Research Rules` (updated: add hybrid retrieval guidance + decisions.md read)
     - `### Partial Analysis File Contents` (updated: add confidence metadata)
   - `## Mode 2: Synthesis` with existing sections
   - `## Completion Contract` (Focused: `DONE: <focus-area> — <summary>` / `ERROR:` ; Synthesis: `DONE: synthesis — <summary>` / `ERROR:`)
   - `## Anti-Drift Anchor` (exact text from design.md table)

## Estimated Effort

Medium

## Completion Checklist

- [x] File created at `NewAgentsAndPrompts/researcher.agent.md`
- [x] Canonical section order followed
- [x] Operating Rules block included with researcher-specific Rule 5
- [x] Retrieval Strategy subsection present with 5-step methodology
- [x] Confidence metadata fields in partial analysis format
- [x] decisions.md read convention in focused research rules
- [x] Completion contract is two-state (DONE/ERROR only)
- [x] Anti-drift anchor is last section with exact text
- [x] No hardcoded project names or technology-specific references
- [x] `detailed thinking on` directive present with experimental comment
