# Task 09: Adversarial Reviewer Agent Definition

## Task Goal

Create `NewAgents/.github/agents/adversarial-reviewer.agent.md` — the parameterized multi-model review agent with YAML verdict output, SQL INSERT, and perspective-diverse `review_focus`.

## depends_on

01, 02

## agent

implementer

## In-Scope

- Agent definition following design.md §Decision 10 template
- Role: parameterized adversarial review (design OR code scope)
- Dispatch parameters:
  - `review_scope`: design | code
  - `model`: gpt-5.3-codex | gemini-3-pro-preview | claude-opus-4.6
  - `review_focus` (v4 H5): security | architecture | correctness
  - `risk_level`: 🟢 | 🟡 | 🔴
  - `verification_evidence_path`: path to verification ledger DB (for code review)
  - `run_id`, `round`: from orchestrator context
- Input: review scope artifacts, verification evidence
- Output:
  - `review-findings/<scope>-<model>.md` — Markdown findings per model (human-readable)
  - `review-verdicts/<scope>.yaml` — YAML verdict summary (v4 H1) with: reviewer_model, review_focus, scope, verdict, findings_count, summary
  - SQL INSERT into `anvil_checks` with `phase='review'`, run_id, round, verdict, severity
- Focus area details (v4 H5):
  - `security`: injection vectors, auth gaps, data exposure, secrets, authorization bypass
  - `architecture`: coupling, scalability boundaries, component responsibilities, API surface
  - `correctness`: edge cases, logic errors, spec compliance, testing gaps, error handling
- Review findings format: Markdown with verdict, confidence, findings (severity/category/description/affected/recommendation/evidence), summary
- Always dispatched 3× in parallel with distinct review_focus values
- Disagreement resolution protocol (from design.md §Decision 7)
- Security blocker policy: any Blocker → pipeline ERROR regardless of other verdicts
- Max 2 review rounds
- Tool access: `read_file`, `list_dir`, `grep_search`, `semantic_search`, `file_search`, `run_in_terminal` (for `git diff --staged` in code review + SQL INSERT), `create_file` (for verdict YAML)
- Tool restriction: read-only for existing files; may only create its own output files
- Completion contract: DONE | ERROR
- Self-verification + anti-drift anchor

## Out-of-Scope

- Schema definitions (in schemas.md)
- Dispatch coordination (orchestrator handles 3× parallel dispatch)
- Model routing mechanism (platform-level concern)
- Model routing fallback (orchestrator-level decision)

## Acceptance Criteria

1. File exists at `NewAgents/.github/agents/adversarial-reviewer.agent.md`
2. Follows agent definition template with all required sections
3. References `review-findings` schema from schemas.md
4. Dispatch parameters documented (review_scope, model, review_focus, risk_level, run_id, round)
5. All 3 review_focus areas documented with specific concern lists (security/architecture/correctness)
6. YAML verdict output format documented matching design.md §Decision 2 (H1)
7. SQL INSERT format documented: phase='review', verdict, severity, run_id, round, check_name pattern
8. Review findings Markdown format documented (verdict, confidence, findings, summary)
9. Security blocker policy: any Blocker finding → verdict='blocker'
10. Tool list matches design.md §Decision 2 (Adversarial Reviewer)
11. Completion contract: DONE | ERROR
12. Self-verification and anti-drift anchor present

## Estimated Effort

Medium

## Test Requirements

- Grep for review_focus with security, architecture, correctness
- Grep for YAML verdict output fields
- Grep for SQL INSERT and anvil_checks
- Grep for Blocker/security policy
- Verify completion contract and anti-drift

## Implementation Steps

1. Read design.md §Decision 2, Adversarial Reviewer agent detail for inputs, outputs, tools, parameters, contract
2. Read design.md §Decision 7 for review design, findings format, disagreement resolution, security blocker policy, review cycling
3. Read design.md §Decision 3, Steps 3b and 7 for dispatch flow with review_focus (H5)
4. Read design.md §Decision 6 for SQL INSERT format (phase='review' column semantics)
5. Reference `Anvil/anvil.agent.md` for adversarial review patterns and SQL verdict insertion
6. Create `NewAgents/.github/agents/adversarial-reviewer.agent.md` with full template structure including:
   - Dispatch parameter definitions with all options
   - Focus area checklists (security/architecture/correctness)
   - Output format for Markdown findings + YAML verdict + SQL INSERT
   - Security blocker policy
   - Review round awareness (round tracking)
7. Self-verify all sections present

## Relevant Context from design.md

- §Decision 2 (lines ~265–310) — Adversarial Reviewer: inputs, outputs, tools, parameters, YAML verdict (H1), focus areas (H5)
- §Decision 7 (lines ~880–970) — Review design, findings format, disagreement resolution, security blocker, review cycling, model routing fallback (C4)
- §Decision 3, Steps 3b/7 (lines ~340–365, 420–445) — Review dispatch with perspective-diverse focus
- §Decision 6 (lines ~760–790) — SQL column semantics for review records

## Completion Checklist

- [x] File created at correct path
- [x] All template sections present
- [x] Dispatch parameters documented (6 parameters)
- [x] 3 review_focus areas documented with checklists
- [x] YAML verdict output format documented
- [x] SQL INSERT format documented
- [x] Security blocker policy present
- [x] Markdown findings format documented
- [x] Schema reference to schemas.md present
- [x] Anti-drift anchor present

## Implementation Notes

- TDD skipped: markdown agent definition — no behavioral code, validated via grep-based acceptance criteria checks
- All 12 acceptance criteria verified via grep searches against the output file
- Agent definition follows §Decision 10 template structure consistent with existing agents (researcher, spec, planner, designer)
