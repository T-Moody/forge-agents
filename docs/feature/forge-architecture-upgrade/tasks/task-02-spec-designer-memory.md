# Task 02: Spec + Designer — Memory Protocol

## Status

DONE

## Agent

implementer

## Depends On

none

## Description

Update `spec.agent.md` and `designer.agent.md` to integrate the memory protocol. Both are sequential agents that read AND write to `memory.md`. The changes are identical in structure: add `memory.md` as first input, add memory-first operating rule, prepend memory read to workflow, append memory write before completion contract.

## Input Files

- `docs/feature/forge-architecture-upgrade/design.md` — §8.1 (Memory Protocol), §8.2 (Spec + Designer specific changes)
- `NewAgentsAndPrompts/spec.agent.md` — current file to modify
- `NewAgentsAndPrompts/designer.agent.md` — current file to modify

## Output File

- `NewAgentsAndPrompts/spec.agent.md`
- `NewAgentsAndPrompts/designer.agent.md`

## Acceptance Criteria

1. [x] Both files list `memory.md` as the FIRST item in their Inputs section: `docs/feature/<feature-slug>/memory.md (read first — operational memory)`
2. [x] Both files have Operating Rule 6 added: memory-first reading rule
3. [x] Both files have Workflow Step 1 prepended: read `memory.md` for orientation (existing steps renumbered)
4. [x] Both files have a penultimate workflow step (before completion contract return) that writes to `memory.md`: Artifact Index, Recent Decisions, Recent Updates
5. [x] All existing functionality preserved (outputs, completion contracts, anti-drift anchors)
6. [x] Memory write entries are constrained to ≤2 sentences each per design.md §1.3

## Implementation Guidance

Apply the following changes identically to both files:

**1. Inputs section:** Add as FIRST input:

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

```
1. Read `memory.md` to load artifact index, recent decisions, lessons learned,
   and recent updates. Use this to orient before reading source artifacts.
```

Renumber all existing workflow steps (old Step 1 → Step 2, etc.).

**4. Workflow — append penultimate step (before completion contract return):**

```
N. Update `memory.md`: append to the appropriate sections:
   - **Artifact Index:** Add path and key sections of the output artifact.
   - **Recent Decisions:** If design/spec decisions were made (≤2 sentences each).
   - **Recent Updates:** Summary of output produced (≤2 sentences).
```

**Designer-specific note:** The designer also reads `design_critical_review.md` during revision cycles. This does not change — the memory protocol is additive.

**Note:** These are markdown agent definition files — TDD does not apply. Modify the files directly per the specifications above.
