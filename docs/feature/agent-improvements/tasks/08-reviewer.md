# Task 08: Write Reviewer Agent

## Task Goal

Create the complete `reviewer.agent.md` file in `NewAgentsAndPrompts/`, rewriting the reviewer to include security review, tiered review depth, read-only enforcement, decision log write convention, quality standard, and self-reflection.

**Output file:** `NewAgentsAndPrompts/reviewer.agent.md`

## depends_on

none

## In-Scope

- Rewrite of `reviewer.agent.md` following canonical structure
- Frontmatter with `name: reviewer` and updated description ("security-aware", "tiered depth", "architectural decision log")
- Role preamble (3 sentences)
- `detailed thinking on` directive
- **Inputs** section (initial-request.md, git diff, entire codebase)
- **Outputs** section (NEW — explicit: review.md, .github/instructions updates, decisions.md append-only)
- **Operating Rules** block (reviewer variant, Rule 5: grep_search, read_file, never modify source)
- **Read-Only Enforcement** (NEW — cannot modify source code, only writes review.md + instructions + decisions.md)
- **Review Depth Tiers** (NEW — Full/Standard/Lightweight with trigger conditions and scope)
- **Workflow** (REWRITTEN — 11-step with security, tiers, decision log, self-reflection)
- **Security Review** (NEW — heuristic pre-scan for large diffs, standard secrets/PII scan for all reviews, OWASP Top 10 for Full tier)
- **Quality Standard** (NEW — "Would a staff engineer approve this code?")
- **review.md Contents** (updated — adds review tier, security findings, architectural decisions)
- **Completion Contract** — THREE-STATE: DONE/NEEDS_REVISION/ERROR (reviewer is one of 3 agents using NEEDS_REVISION)
- **Anti-Drift Anchor**

## Out-of-Scope

- Modifying files in `.github/`
- Modifying source code (reviewer is read-only)
- OWASP review for non-security-sensitive changes (Standard/Lightweight tiers skip deep OWASP)

## Acceptance Criteria

1. File exists at `NewAgentsAndPrompts/reviewer.agent.md` and is non-empty
2. Contains Security Review section with:
   - Standard secrets/PII scan (patterns: password, secret, api_key, token, Bearer, etc.) for ALL reviews
   - OWASP Top 10 checklist for Full tier only (10 items)
   - Heuristic pre-scan for large diffs (20+ files)
3. Contains Review Depth Tiers: Full (security-sensitive), Standard (business logic), Lightweight (docs/config)
4. Contains Read-Only Enforcement prohibiting source code modification
5. Workflow includes decision log step: append significant decisions to `decisions.md`, create if not exists
6. Workflow includes self-reflection step (all files reviewed, comments actionable, security scan completed)
7. Contains Quality Standard: "Would a staff engineer approve this code?"
8. Completion contract is THREE-STATE: DONE/NEEDS_REVISION/ERROR
9. DONE format: `review complete — <tier> review, <N> issues (<M> blocking)`
10. NEEDS_REVISION routes to implementer(s) for lightweight fixes (max 1 loop); ERROR triggers full replan
11. review.md Contents includes review tier, security findings, architectural decisions
12. Anti-drift anchor: "You are the **Reviewer**. You review code for quality, correctness, and security. You never modify source code. You write review findings only. Stay as reviewer."

## Test Requirements

1. **Security review test (TS-6):** Contains OWASP reference and secrets scanning patterns
2. **Review tiers test:** Contains Full/Standard/Lightweight tiers with triggers
3. **Decision log test:** Workflow mentions decisions.md with append-only convention
4. **Three-state test:** Completion contract includes NEEDS_REVISION with implementer routing
5. **Cross-agent consistency (TS-14):** DONE/ERROR format matches orchestrator expectations
6. **Error handling test (TS-15):** Operating Rules contain error handling categories

## Implementation Steps

1. Read design.md § Per-Agent Design → 9. `reviewer.agent.md` for target state and ALL content specifications
2. Read design.md § Cross-Cutting Patterns for:
   - Pattern 3: Completion Contract — use THREE-STATE for reviewer
   - Pattern 5: decisions.md Lifecycle — reviewer is the ONLY writer
3. Create `NewAgentsAndPrompts/reviewer.agent.md` with:
   - YAML frontmatter
   - `# Reviewer Agent Workflow` title
   - Role preamble
   - `detailed thinking on` directive
   - `## Inputs` (initial-request.md, git diff, entire codebase)
   - `## Outputs` (review.md, .github/instructions, decisions.md)
   - `## Operating Rules` (reviewer variant)
   - `## Read-Only Enforcement` (from design.md)
   - `## Review Depth Tiers` (Full/Standard/Lightweight table from design.md)
   - `## Workflow` (11-step from design.md)
   - `## Security Review` with:
     - All Reviews subsection (heuristic pre-scan + standard scan)
     - Full Tier Only subsection (OWASP Top 10)
   - `## Quality Standard` ("Would a staff engineer approve this code?")
   - `## review.md Contents` (updated)
   - `## Completion Contract` (THREE-STATE with routing guidance)
   - `## Anti-Drift Anchor`

## Estimated Effort

Medium

## Completion Checklist

- [x] File created at `NewAgentsAndPrompts/reviewer.agent.md`
- [x] Security Review section with secrets scan + OWASP
- [x] Review Depth Tiers (Full/Standard/Lightweight)
- [x] Read-Only Enforcement section
- [x] Decision log write convention in workflow
- [x] Quality Standard present
- [x] Self-reflection step in workflow
- [x] Completion contract is three-state (DONE/NEEDS_REVISION/ERROR)
- [x] NEEDS_REVISION routes to implementers, ERROR triggers replan
- [x] review.md Contents includes tier, security findings, decisions
- [x] Anti-drift anchor with exact text
- [x] Heuristic pre-scan for large diffs (20+ files)
