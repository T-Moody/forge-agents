# Task 06: Researcher — Remove Synthesis Mode, Add Isolated Memory

## Task Goal

Remove the researcher's synthesis mode (Mode 2) entirely and add isolated memory output for the focused mode, transforming the researcher into a focused-only agent that writes findings to `memory/researcher-<focus>.mem.md`.

## depends_on

none

## agent

implementer

## In-Scope

- Remove synthesis mode (Mode 2) entirely: all Mode 2 inputs, outputs, workflow steps, completion contract, `analysis.md` contents template
- Remove `analysis.md` from all outputs references
- Remove synthesis mode inputs (research/\*.md as merge inputs for synthesis)
- Update description/role to indicate focused mode only (no modes distinction needed)
- Add `memory/researcher-<focus>.mem.md` to Outputs section (e.g., `memory/researcher-architecture.mem.md`)
- Add "Write Isolated Memory" workflow step as final step before completion contract
- Update Anti-Drift Anchor: replace any "write to `memory.md`" references with "write only to your isolated memory file (`memory/researcher-<focus>.mem.md`), never to shared `memory.md`"
- Remove the "Update `memory.md`" workflow step (synthesis mode had this) — replace with the new isolated memory write step

## Out-of-Scope

- Modifying other agent files that reference `analysis.md` (handled by tasks 04, 07, 08)
- Modifying the orchestrator (handled by tasks 02–05)

## Acceptance Criteria

- AC-4: `researcher.agent.md` contains no "Mode 2", "Synthesis", "synthesis mode", or `analysis.md` references
- AC-5: Outputs section lists `memory/researcher-<focus>.mem.md`; workflow contains "Write Isolated Memory" step
- AC-16: Anti-Drift Anchor references isolated memory file, not shared `memory.md` write
- AC-19: Standard section ordering preserved (YAML frontmatter → Title → Role → Prohibitions → Inputs → Outputs → Operating Rules → Workflow → Completion Contract → Anti-Drift Anchor)
- AC-20: Completion contract format unchanged (DONE/ERROR)

## Estimated Effort

Medium

## Test Requirements

TDD skipped: markdown-only task, no behavioral code.

## Implementation Steps

1. Read `researcher.agent.md` fully to identify all synthesis mode content locations
2. Search for "Mode 2", "Synthesis", "synthesis", "analysis.md" to identify all removal targets
3. Remove Mode 2 section header and all synthesis mode workflow steps
4. Remove `analysis.md` from Outputs section
5. Remove `analysis.md` contents template (if present)
6. Remove synthesis mode completion contract variation
7. Remove synthesis-specific inputs (research/\*.md as merge/synthesis inputs)
8. Update role/description to remove any modes distinction — the researcher is focused-only
9. Add to **Outputs section**: `docs/feature/<feature-slug>/memory/researcher-<focus>.mem.md` (isolated memory)
10. Add **"Write Isolated Memory" workflow step** as final step before completion contract:
    ```
    ### N. Write Isolated Memory
    Write key findings to `memory/researcher-<focus>.mem.md`:
    - Status: completion status (DONE/ERROR)
    - Key Findings: ≤5 bullet points summarizing primary findings
    - Highest Severity: N/A
    - Decisions Made: any decisions taken (omit if none)
    - Artifact Index: list of output file paths with section-level pointers (§Section Name) and brief relevance notes
    ```
11. Replace any "Update `memory.md`" step with the new isolated memory step
12. Update **Anti-Drift Anchor**: "You write only to your isolated memory file (`memory/researcher-<focus>.mem.md`), never to shared `memory.md`."
13. Verify no "Mode 2", "Synthesis", "synthesis", "analysis.md" strings remain
14. Verify section ordering matches convention

## Completion Checklist

- [x] No "Mode 2" string in file
- [x] No "Synthesis" or "synthesis mode" strings in file
- [x] No "analysis.md" string in file
- [x] Outputs section includes `memory/researcher-<focus>.mem.md`
- [x] Workflow has "Write Isolated Memory" step with correct template
- [x] Old "Update `memory.md`" step removed (if existed for synthesis mode)
- [x] Anti-Drift Anchor references isolated memory, not shared `memory.md`
- [x] Completion contract format unchanged (DONE/ERROR)
- [x] Section ordering matches convention
- [x] Role description reflects focused mode only
