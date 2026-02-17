# Task 01: Researcher — 4th Focus Area + Memory Protocol

## Agent

implementer

## Depends On

none

## Description

Update `researcher.agent.md` to add the 4th research focus area (`patterns`) and integrate the memory protocol. The researcher runs in two modes: focused (parallel, 4 instances) and synthesis (sequential). In focused mode, the researcher reads `memory.md` but does NOT write to it (parallel agent). In synthesis mode, the researcher reads and writes `memory.md` (sequential agent).

## Input Files

- `docs/feature/forge-architecture-upgrade/design.md` — §6 (Research Expansion), §8.1 (Memory Protocol), §8.2 (Researcher-specific changes)
- `NewAgentsAndPrompts/researcher.agent.md` — current file to modify

## Output File

- `NewAgentsAndPrompts/researcher.agent.md`

## Acceptance Criteria

1. [x] Focus Area table contains 4 rows: `architecture`, `impact`, `dependencies`, `patterns`
2. [x] The `patterns` focus area scope matches design.md §6.1: "Testing strategy, code conventions, developer experience patterns, error handling patterns, operational concerns, existing reusable patterns"
3. [x] `memory.md` is listed as the FIRST input in both Focused Mode Inputs and Synthesis Mode Inputs sections
4. [x] Operating Rule 6 added: memory-first reading rule (read `memory.md` before accessing any artifact; use Artifact Index for navigation; fallback if missing)
5. [x] Focused mode workflow: Step 1 reads `memory.md` → existing steps renumbered. NO memory write step (parallel agent)
6. [x] Synthesis mode workflow: Step 1 reads `memory.md` → existing steps renumbered. Penultimate step writes to memory (Artifact Index + Recent Updates) — sequential agent, safe to write
7. [x] All existing functionality is preserved (retrieval strategy, synthesis rules, completion contracts, anti-drift anchor)
8. [x] No other structural changes beyond the additions above

## Status

Complete — TDD skipped: markdown agent definition file, no behavioral code.

## Implementation Guidance

**4th Focus Area (design.md §6.2):**

Add row to the Focus Area table in the "Focused Research" section:

```
| **patterns** | Testing strategy, code conventions, developer experience patterns, error handling patterns, operational concerns |
```

No changes needed to synthesis mode instructions — the existing rule "Read ALL partial research files from the `research/` directory" already accommodates a 4th `patterns.md` file automatically.

**Memory Protocol (design.md §8.1 + §8.2):**

1. **Inputs (both modes):** Add `docs/feature/<feature-slug>/memory.md` as the FIRST item in both the Focused Mode and Synthesis Mode input lists. Add note `(read first — operational memory)`.

2. **Operating Rule 6:** Add after Rule 5:

   ```
   6. **Memory-first reading:** Read `memory.md` FIRST before accessing any artifact.
      Use the Artifact Index to navigate directly to relevant sections rather than
      reading full artifacts. If `memory.md` is missing, log a warning and proceed
      with direct artifact reads.
   ```

3. **Focused mode workflow:** Prepend new Step 1: "Read `memory.md` to load artifact index, recent decisions, lessons learned, and recent updates. Use this to orient before reading source artifacts." Renumber existing steps. Do NOT add a memory write step — focused researchers are parallel agents and must NOT write to `memory.md`.

4. **Synthesis mode workflow:** Prepend new Step 1 (same as above). Add penultimate step before completion contract: "Update `memory.md`: append to Artifact Index (path and key sections of `analysis.md`) and Recent Updates (summary of synthesis output). Each entry ≤2 sentences." Renumber existing steps.

**Note:** These are markdown agent definition files — TDD does not apply. Create/modify the file directly per the specifications above.
