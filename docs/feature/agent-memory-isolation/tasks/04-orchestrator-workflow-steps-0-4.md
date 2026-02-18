# Task 04: Orchestrator — Workflow Steps 0–4

## Task Goal

Update orchestrator Workflow Steps 0 through 4 to reflect the isolated memory model: add `memory/` directory creation at Step 0, remove Step 1.2 (synthesis), update Step 2 and 3 inputs to memory-first pattern, replace Step 3b.2 (CT aggregator invocation) with orchestrator CT evaluation, and add memory merge sub-steps.

## depends_on

03

## agent

implementer

## In-Scope

- **Step 0**: Add creation of `memory/` directory alongside `initial-request.md` and `memory.md`
- **Step 1.1**: Add merge sub-step (1.1m) after research agents return — orchestrator reads 4 researcher memories, merges into `memory.md`, prunes
- **Step 1.2**: Remove entirely (synthesis researcher invocation). Update subsequent step numbering references if needed
- **Step 1.1a / 1.2a**: Update approval gate reference from "after Step 1.2" to "after Step 1.1"
- **Step 2**: Update spec inputs — replace `analysis.md` with researcher memory files as primary inputs + `research/*.md` for selective reads. Add merge sub-step (2m)
- **Step 3**: Update designer inputs — replace `analysis.md` with spec + researcher memory files as primary inputs + selective reads. Add merge sub-step (3m)
- **Step 3b.1**: Keep CT sub-agent dispatch (no change)
- **Step 3b.2**: Replace CT aggregator invocation with orchestrator CT evaluation (reference the CT decision flow added in task 03). Add merge sub-step (3bm)
- **Step 3b.3**: Update CT NEEDS_REVISION handling — designer reads CT memories → targeted reads of `ct-review/ct-*.md` sections (not `design_critical_review.md`). Replace "forward Unresolved Tensions as planning constraints" with "forward all High/Critical findings from individual CT artifacts as planning constraints"
- **Step 4**: Update planner inputs — replace `analysis.md` with designer + spec memory files as primary inputs + selective reads of `design.md`, `feature.md`, `research/*.md` sections. Remove `design_critical_review.md`. Add CT memories → selective `ct-review/ct-*.md` reads if present. Add merge sub-step (4m) and prune

## Out-of-Scope

- Steps 5–7 (handled by task 05)
- Cluster decision logic definitions (already done in task 03)
- Global Rules (already done in task 02)

## Acceptance Criteria

- AC-18: No Step 1.2 or synthesis researcher invocation exists
- AC-6 (partial): Merge sub-steps present after each cluster/agent completion in Steps 0–4
- AC-3 (partial): No `analysis.md` or `design_critical_review.md` references in Steps 0–4
- AC-2 (partial): No aggregator names in Steps 0–4
- Step 0 creates `memory/` directory
- Step 2 inputs reference researcher memories as primary, `research/*.md` for selective reads
- Step 3 inputs reference spec + researcher memories, selective artifact reads
- Step 3b.2 references orchestrator CT evaluation (not ct-aggregator invocation)
- Step 3b.3 references CT memories and individual `ct-review/ct-*.md` files (not `design_critical_review.md`)
- Step 4 inputs reference designer + spec memories, selective reads, CT memories if present

## Estimated Effort

Medium

## Test Requirements

TDD skipped: markdown-only task, no behavioral code.

## Implementation Steps

1. Read `orchestrator.agent.md` Steps 0–4 (approximately lines 130–220)
2. **Step 0**: Add `memory/` directory creation line (e.g., "Create `docs/feature/<feature-slug>/memory/` directory")
3. **Step 1.1**: Add merge sub-step after research agents return: "Read `memory/researcher-architecture.mem.md`, `memory/researcher-impact.mem.md`, `memory/researcher-dependencies.mem.md`, `memory/researcher-patterns.mem.md` → merge into `memory.md` → prune"
4. **Step 1.2**: Delete the entire Step 1.2 section (synthesis researcher invocation). If there's a Step 1.2 approval gate, move it to after Step 1.1
5. **Step 2**: Replace `analysis.md` input with "Read researcher memories (`memory/researcher-*.mem.md`) → consult artifact indexes → targeted reads of `research/*.md` sections". Add merge sub-step after spec returns
6. **Step 3**: Replace `analysis.md` input with "Read spec memory (`memory/spec.mem.md`) + researcher memories → consult artifact indexes → targeted reads of `feature.md` + `research/*.md` sections". Add merge sub-step
7. **Step 3b.2**: Replace CT aggregator invocation with "Apply CT Cluster Decision (see CT decision flow above)". Add merge sub-step: "Read 4 CT memories → merge into `memory.md`"
8. **Step 3b.3**: Replace `design_critical_review.md` references with CT memories and individual `ct-review/ct-*.md` files. Replace "forward Unresolved Tensions" with "forward all High/Critical findings from individual CT artifacts as planning constraints"
9. **Step 4**: Replace `analysis.md` with designer + spec memories. Remove `design_critical_review.md`. Add selective reads of `design.md`, `feature.md`, `research/*.md` via memory artifact indexes. Add CT memories if present. Add merge sub-step + prune
10. Search modified sections for remaining references to `analysis.md`, `design_critical_review.md`, `ct-aggregator`, `synthesis`, "Step 1.2", "Unresolved Tensions"

## Status

COMPLETED

## Completion Checklist

- [x] Step 0 creates `memory/` directory
- [x] Step 1.2 is fully removed (no synthesis invocation)
- [x] Merge sub-steps present after Steps 1.1, 2, 3, 3b, 4
- [x] Prune checkpoints at Steps 1.1 and 4 (per updated Global Rule 6) — also at Step 2m
- [x] No `analysis.md` references in Steps 0–4
- [x] No `design_critical_review.md` references in Steps 0–4
- [x] No `ct-aggregator` references in Steps 0–4
- [x] No "Unresolved Tensions" references — replaced with "High/Critical findings from individual CT artifacts"
- [x] Step 3b.2 references orchestrator CT evaluation, not aggregator
- [x] Approval gate references Step 1.1, not Step 1.2
- [x] Standard section ordering preserved

## Findings

- TDD skipped: markdown-only task, no behavioral code.
- Remaining references to `analysis.md`, `design_critical_review.md`, aggregators, and `synthesis` exist in the bottom tables (NEEDS_REVISION Routing Table, Orchestrator Expectations Per Agent, Parallel Execution Summary) — these are out of scope for this task and are covered by task 05.
