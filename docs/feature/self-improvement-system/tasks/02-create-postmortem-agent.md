# Task 02: Create PostMortem Agent Definition

**Task Goal:** Create `.github/agents/post-mortem.agent.md` — a new non-blocking agent that produces quantitative cross-agent metrics from evaluations and telemetry at Step 8.

**depends_on:** none

**agent:** implementer

---

## In-Scope

- Create new file `.github/agents/post-mortem.agent.md`
- Full chatagent format: YAML frontmatter, title/role, Inputs, Outputs, Operating Rules (6), Reading Strategy, File Boundaries, Read-Only Enforcement, Workflow, Completion Contract, Anti-Drift Anchor
- Memory-first reading pattern with targeted evaluation file reads
- Quantitative-only output (no improvement_recommendations, no knowledge-suggestions.md)
- Two-state completion contract (DONE/ERROR only, no NEEDS_REVISION)
- Non-blocking designation
- Self-verification step checking R-Knowledge boundary compliance

## Out-of-Scope

- Modifying the orchestrator to dispatch PostMortem (that's Task 10)
- Defining the evaluation schema (that's Task 01)
- Modifying R-Knowledge agent (explicitly excluded per AC-3/AC-12)
- Any runtime execution — this is an agent definition file

---

## Acceptance Criteria

1. File exists at `.github/agents/post-mortem.agent.md`
2. YAML frontmatter: `name: post-mortem`, `description` mentions non-blocking and quantitative
3. Inputs section includes: orchestrator dispatch context, `memory/*.mem.md`, `artifact-evaluations/*.md`, `memory.md`, `evaluation-schema.md`
4. Outputs section includes: `agent-metrics/<run-date>-run-log.md`, `post-mortems/<run-date>-post-mortem.md`, `memory/post-mortem.mem.md`
5. 6 Operating Rules present with agent-specific details (corrupted YAML handling, memory-first reading, tool preferences)
6. Reading Strategy section describes the memory-first pattern with 5 steps
7. File Boundaries section explicitly lists the 3 output files and prohibits writing elsewhere
8. Read-Only Enforcement section present
9. Workflow has 11 numbered steps including self-verification (step 9)
10. Self-verification checks: all evaluations processed, all telemetry accounted for, quantitative-only, no R-Knowledge overlap
11. Completion Contract: two-state (DONE/ERROR only), no NEEDS_REVISION, states non-blocking
12. Anti-Drift Anchor: mentions "quantitative metrics only", does NOT mention improvement_recommendations or knowledge-suggestions.md
13. Post-mortem report YAML schema in workflow includes: `run_date`, `feature_slug`, `summary`, `recurring_issues`, `bottlenecks`, `most_common_missing_information`, `agent_accuracy_scores` — NO `improvement_recommendations` field
14. Run-log YAML format matches design.md §Run-Log File Format
15. Append-only: sequence suffix collision avoidance for same-day re-runs

---

## Estimated Effort

Medium (~250 lines)

---

## Test Requirements (TDD Fallback — Configuration-Only)

1. **Verify file exists** at `.github/agents/post-mortem.agent.md`
2. **Verify frontmatter:** grep for `name: post-mortem`
3. **Verify Inputs section:** grep for `artifact-evaluations/*.md`, `memory/*.mem.md`, `evaluation-schema.md`
4. **Verify Outputs section:** grep for `run-log.md`, `post-mortem.md`, `post-mortem.mem.md`
5. **Verify no NEEDS_REVISION:** grep confirms absence of `NEEDS_REVISION` in Completion Contract
6. **Verify no improvement_recommendations:** grep confirms absence of `improvement_recommendations` anywhere in file
7. **Verify no knowledge-suggestions:** grep confirms absence of `knowledge-suggestions` anywhere in file
8. **Verify self-verification step:** grep for "Self-verification" or "self-verification" in workflow
9. **Verify Anti-Drift Anchor:** grep for "quantitative metrics only"
10. **Verify Read-Only Enforcement section exists**

---

## Implementation Steps

1. Read design.md §PostMortem Agent Definition (Revised) for the complete agent specification — this provides nearly all the text
2. Read design.md §Data Models & DTOs → §Run-Log File Format for the run-log YAML structure
3. Read design.md §Data Models & DTOs → §Post-Mortem Report Schema for the report YAML structure (ensure `improvement_recommendations` is NOT included)
4. Create `.github/agents/post-mortem.agent.md` with all sections from design.md:
   - YAML frontmatter (`name: post-mortem`)
   - Title & Role section
   - Inputs section
   - Outputs section (with `<run-date>` placeholders and sequence suffix note)
   - 6 Operating Rules
   - Reading Strategy (memory-first pattern, 5 steps)
   - File Boundaries
   - Read-Only Enforcement
   - Workflow (11 steps: extract telemetry, write run-log, read memories, discover evaluations, extract metrics, compute scores, identify recurring issues, identify bottlenecks, self-verification, write report, write memory)
   - Completion Contract (two-state: DONE/ERROR)
   - Anti-Drift Anchor (quantitative only, not R-Knowledge scope)
5. Verify: no `improvement_recommendations` anywhere in the file
6. Verify: no `knowledge-suggestions` anywhere in the file
7. Verify: Completion Contract says DONE/ERROR only, explicitly non-blocking
8. Verify: post-mortem report schema in the workflow/data-models section has no `improvement_recommendations` field

---

## Completion Checklist

- [x] File created at `.github/agents/post-mortem.agent.md`
- [x] YAML frontmatter with `name: post-mortem`
- [x] Follows standard chatagent format (frontmatter, Inputs, Outputs, 6 Operating Rules, Workflow, Completion Contract, Anti-Drift Anchor)
- [x] Memory-first Reading Strategy section present with 5 steps
- [x] File Boundaries and Read-Only Enforcement sections present
- [x] Self-verification step checks R-Knowledge boundary compliance
- [x] Two-state completion contract (DONE/ERROR only)
- [x] No `improvement_recommendations` field anywhere
- [x] No `knowledge-suggestions.md` or `decisions.md` in outputs
- [x] Anti-Drift Anchor says "quantitative metrics only"
- [x] Sequence suffix collision avoidance documented
- [x] No existing files were modified

TDD skipped: agent definition markdown file creation
