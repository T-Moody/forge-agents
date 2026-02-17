# Task 03: Planner — Memory Protocol

## Agent

implementer

## Depends On

none

## Description

Update `planner.agent.md` to integrate the memory protocol. The planner is a sequential agent that reads AND writes to `memory.md`. Add `memory.md` as first input, add memory-first operating rule, prepend memory read to workflow, append memory write before completion contract.

## Input Files

- `docs/feature/forge-architecture-upgrade/design.md` — §8.1 (Memory Protocol), §8.2 (Planner-specific changes)
- `NewAgentsAndPrompts/planner.agent.md` — current file to modify

## Output File

- `NewAgentsAndPrompts/planner.agent.md`

## Acceptance Criteria

1. `memory.md` listed as FIRST input: `docs/feature/<feature-slug>/memory.md (read first — operational memory)`
2. Operating Rule 6 added: memory-first reading rule
3. Workflow Step 1 prepended: read `memory.md` for orientation (existing steps renumbered)
4. Penultimate workflow step added (before completion contract): write to `memory.md` — Artifact Index, Recent Decisions, Recent Updates
5. All existing functionality preserved (mode detection, planning principles, task size limits, plan validation, pre-mortem, completion contract, anti-drift anchor)
6. Memory write entries constrained to ≤2 sentences each

## Implementation Guidance

**1. Inputs section:** Add as FIRST input (before `initial-request.md`):

```
- docs/feature/<feature-slug>/memory.md (read first — operational memory)
```

**2. Operating Rule 6:** Add after Rule 5:

```
6. **Memory-first reading:** Read `memory.md` FIRST before accessing any artifact.
   Use the Artifact Index to navigate directly to relevant sections rather than
   reading full artifacts. If `memory.md` is missing, log a warning and proceed
   with direct artifact reads.
```

**3. Workflow — prepend Step 1:**
Insert before existing Step 1 (Detect planning mode):

```
1. Read `memory.md` to load artifact index, recent decisions, lessons learned,
   and recent updates. Use this to orient before reading source artifacts.
```

Renumber all existing workflow steps (old Step 1 → Step 2, etc.).

**4. Workflow — append penultimate step (before "Create one task file per task"):**

```
N. Update `memory.md`: append to the appropriate sections:
   - **Artifact Index:** Add path and key sections of `plan.md` and task files.
   - **Recent Decisions:** Planning decisions made (task decomposition rationale, wave structure) — ≤2 sentences each.
   - **Recent Updates:** Summary of planning output produced (≤2 sentences).
```

**Note:** These are markdown agent definition files — TDD does not apply.
