# Task 07: Spec + Designer — Memory-First Inputs, Isolated Memory

## Task Goal

Update `spec.agent.md` and `designer.agent.md` to use the memory-first reading pattern (read upstream memories → consult artifact indexes → targeted artifact reads), add isolated memory outputs, and remove references to `analysis.md` and `design_critical_review.md`.

## depends_on

none

## agent

implementer

## In-Scope

### spec.agent.md

- **Inputs**: Remove `analysis.md`. Add primary: `memory/researcher-architecture.mem.md`, `memory/researcher-impact.mem.md`, `memory/researcher-dependencies.mem.md`, `memory/researcher-patterns.mem.md`. Add selective: `research/architecture.md`, `research/impact.md`, `research/dependencies.md`, `research/patterns.md` (read only sections referenced by memory artifact indexes)
- **Outputs**: Add `docs/feature/<feature-slug>/memory/spec.mem.md`
- **Workflow**: Replace "Update `memory.md`" step with "Write Isolated Memory" step. Add memory-first reading workflow: read researcher memories → consult artifact indexes → targeted reads of `research/*.md` sections
- **Anti-Drift Anchor**: Update to reference isolated memory file

### designer.agent.md

- **Inputs**: Remove `analysis.md`. Remove `design_critical_review.md (if exists — read during revision cycle)`. Add primary: `memory/spec.mem.md`, `memory/researcher-*.mem.md`. Add selective: `research/*.md` (sections per artifact indexes). Add revision mode: `memory/ct-*.mem.md` (if exists) → selective reads of `ct-review/ct-*.md` sections
- **Outputs**: Add `docs/feature/<feature-slug>/memory/designer.mem.md`
- **Workflow**: Replace "Update `memory.md`" step with "Write Isolated Memory" step. Add memory-first reading workflow: read spec + researcher memories → artifact indexes → targeted reads. In revision mode: read CT memories → targeted reads of `ct-review/ct-*.md` sections
- **Anti-Drift Anchor**: Update to reference isolated memory file

## Out-of-Scope

- Planner updates (task 08)
- Orchestrator updates (tasks 02–05)
- Modifying research or CT files

## Acceptance Criteria

- AC-3: No `analysis.md` or `design_critical_review.md` references in either file
- AC-5: Both files list isolated memory in Outputs and have "Write Isolated Memory" workflow step
- AC-10: Both list upstream memory files as primary inputs + research files for selective reads
- AC-11: Designer lists `memory/ct-*.mem.md` and `ct-review/ct-*.md` for revision mode (not `design_critical_review.md`)
- AC-16: Anti-Drift Anchors reference isolated memory, no aggregator references
- AC-19: Section ordering preserved
- AC-20: Completion contracts unchanged

## Estimated Effort

Medium

## Test Requirements

TDD skipped: markdown-only task, no behavioral code.

## Implementation Steps

1. Read `spec.agent.md` to identify Inputs, Outputs, Workflow, Anti-Drift Anchor sections
2. **spec.agent.md Inputs**: Remove `analysis.md` line. Add researcher memory files as primary inputs. Add research files for selective reads
3. **spec.agent.md Outputs**: Add `docs/feature/<feature-slug>/memory/spec.mem.md (isolated memory)`
4. **spec.agent.md Workflow**: Add memory-first reading step: "Read researcher memories (`memory/researcher-*.mem.md`) → review key findings and artifact indexes → identify relevant sections of each `research/*.md` file → read only those sections"
5. **spec.agent.md Workflow**: Replace "Update `memory.md`" step with "Write Isolated Memory" step (Status, Key Findings, Highest Severity: N/A, Decisions Made, Artifact Index with §Section pointers)
6. **spec.agent.md Anti-Drift Anchor**: Replace memory write references with "You write only to your isolated memory file (`memory/spec.mem.md`), never to shared `memory.md`"
7. Read `designer.agent.md` to identify same sections
8. **designer.agent.md Inputs**: Remove `analysis.md`. Remove `design_critical_review.md`. Add spec + researcher memory files as primary. Add research files for selective reads. Add revision mode: CT memories → selective CT review reads
9. **designer.agent.md Outputs**: Add `docs/feature/<feature-slug>/memory/designer.mem.md (isolated memory)`
10. **designer.agent.md Workflow**: Add memory-first reading steps for both normal and revision mode. Replace "Update `memory.md`" with "Write Isolated Memory"
11. **designer.agent.md Anti-Drift Anchor**: Update to reference isolated memory
12. Verify no `analysis.md` or `design_critical_review.md` strings remain in either file

## Completion Checklist

- [x] spec.agent.md: no `analysis.md` references
- [x] spec.agent.md: Inputs list researcher memory files + research files for selective reads
- [x] spec.agent.md: Outputs include `memory/spec.mem.md`
- [x] spec.agent.md: Workflow has memory-first reading + "Write Isolated Memory" step
- [x] spec.agent.md: Anti-Drift references isolated memory
- [x] designer.agent.md: no `analysis.md` or `design_critical_review.md` references
- [x] designer.agent.md: Inputs list spec + researcher memories (primary) + CT memories for revision
- [x] designer.agent.md: Outputs include `memory/designer.mem.md`
- [x] designer.agent.md: Workflow has memory-first reading (normal + revision mode) + "Write Isolated Memory" step
- [x] designer.agent.md: Anti-Drift references isolated memory
- [x] Both files preserve section ordering and completion contract format
