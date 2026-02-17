# Task 04: Implementer + Documentation Writer — Memory Protocol

## Agent

implementer

## Depends On

none

## Description

Update `implementer.agent.md` and `documentation-writer.agent.md` to integrate the memory protocol. The implementer is a parallel agent (reads memory, does NOT write). The documentation writer is effectively sequential within implementation waves (reads AND writes memory).

## Input Files

- `docs/feature/forge-architecture-upgrade/design.md` — §8.1 (Memory Protocol), §8.2 (Implementer + Documentation Writer specific changes)
- `NewAgentsAndPrompts/implementer.agent.md` — current file to modify
- `NewAgentsAndPrompts/documentation-writer.agent.md` — current file to modify

## Output File

- `NewAgentsAndPrompts/implementer.agent.md`
- `NewAgentsAndPrompts/documentation-writer.agent.md`

## Status

Complete

## Acceptance Criteria

1. [x] Both files list `memory.md` as FIRST input: `docs/feature/<feature-slug>/memory.md (read first — operational memory)`
2. [x] Both files have Operating Rule 6: memory-first reading rule
3. [x] Both files have Workflow Step 1 prepended: read `memory.md` for orientation (existing steps renumbered)
4. [x] **Implementer:** Does NOT have a memory write step (parallel agent — issues captured in task output files; orchestrator extracts Lessons Learned between waves)
5. [x] **Documentation Writer:** Has a penultimate memory write step (Artifact Index + Recent Updates) — sequential agent, safe to write
6. [x] All existing functionality preserved (TDD workflow, code quality principles, security rules for implementer; read-only enforcement, capabilities for documentation writer)

## Notes

- TDD skipped: markdown agent definition files, no behavioral code changes
- Implementer: Step 0 prepended to TDD Workflow (not Step 1, to preserve TDD step numbering starting at 1)
- Documentation Writer: existing workflow steps renumbered 2-8 (was 1-6), memory write inserted as step 7 (penultimate)

## Implementation Guidance

### Implementer (`implementer.agent.md`)

**1. Inputs section:** Add as FIRST input under `## Inputs (STRICT)`:

```
- docs/feature/<feature-slug>/memory.md (read first — operational memory)
```

Keep the existing note: "You MUST NOT read: plan.md"

**2. Operating Rule 6:** Add after Rule 5:

```
6. **Memory-first reading:** Read `memory.md` FIRST before accessing any artifact.
   Use the Artifact Index to navigate directly to relevant sections rather than
   reading full artifacts. If `memory.md` is missing, log a warning and proceed
   with direct artifact reads.
```

**3. TDD Workflow — prepend Step 0 before "1. Understand":**
Add a step before the TDD workflow:

```
0. Read `memory.md` to load artifact index, recent decisions, lessons learned,
   and recent updates. Use this context to inform implementation decisions.
```

Note: Do NOT add a memory write step. Implementers are parallel agents. Issues and lessons are captured in the task output file. The orchestrator extracts Lessons Learned entries from task outputs between implementation waves and appends them to `memory.md`.

### Documentation Writer (`documentation-writer.agent.md`)

**1. Inputs section:** Add as FIRST input under `## Inputs (STRICT)`:

```
- docs/feature/<feature-slug>/memory.md (read first — operational memory)
```

**2. Operating Rule 6:** Same as implementer (add after Rule 5).

**3. Workflow — prepend Step 1:**

```
1. Read `memory.md` to load artifact index, recent decisions, lessons learned,
   and recent updates. Use this to orient before reading source artifacts.
```

Renumber existing steps.

**4. Workflow — append penultimate step (before completion contract):**

```
N. Update `memory.md`: append to the appropriate sections:
   - **Artifact Index:** Add path and key sections of the documentation produced.
   - **Recent Updates:** Summary of documentation output (≤2 sentences).
```

**Note:** These are markdown agent definition files — TDD does not apply.
