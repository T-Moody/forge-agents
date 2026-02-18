# Task 11: CT Sub-Agents — Isolated Memory (Batch 1)

## Task Goal

Add isolated memory output, "Write Isolated Memory" workflow step, and updated Anti-Drift Anchors to `ct-security.agent.md`, `ct-scalability.agent.md`, and `ct-maintainability.agent.md`.

## depends_on

none

## agent

implementer

## In-Scope

For each of the 3 files, apply the universal sub-agent changes:

1. **Outputs**: Add `docs/feature/<feature-slug>/memory/ct-<domain>.mem.md (isolated memory)` (e.g., `memory/ct-security.mem.md`)
2. **Workflow**: Add "Write Isolated Memory" step as final step before completion contract. CT agents use `Highest Severity: Critical/High/Medium/Low` (CT taxonomy). The template:
   ```
   ### N. Write Isolated Memory
   Write key findings to `memory/ct-<domain>.mem.md`:
   - Status: DONE/ERROR with one-line summary
   - Key Findings: ≤5 bullet points summarizing primary findings
   - Highest Severity: Critical/High/Medium/Low (highest severity finding)
   - Decisions Made: (omit if none)
   - Artifact Index: ct-review/ct-<domain>.md — §Section pointers with brief relevance notes
   ```
3. **"No Memory Write" step** (if exists): Replace with the new "Write Isolated Memory" step
4. **Anti-Drift Anchor**: Replace "You do NOT write to `memory.md`" (or equivalent) with "You write only to your isolated memory file (`memory/ct-<domain>.mem.md`), never to shared `memory.md`." Remove any aggregator references

## Out-of-Scope

- `ct-strategy.agent.md` (handled in task 12)
- Modifying CT analysis methodology
- Orchestrator CT decision logic (task 03)

## Acceptance Criteria

- AC-5: All 3 files list isolated memory in Outputs and have "Write Isolated Memory" workflow step
- AC-16: All 3 Anti-Drift Anchors reference isolated memory, no aggregator references
- AC-19: Section ordering preserved in all 3 files
- AC-20: Completion contracts unchanged (DONE/ERROR)
- `Highest Severity` field uses CT taxonomy (Critical/High/Medium/Low)

## Estimated Effort

Low

## Test Requirements

TDD skipped: markdown-only task, no behavioral code.

## Implementation Steps

1. Read `ct-security.agent.md` Outputs, Workflow, Anti-Drift Anchor sections
2. Apply changes to `ct-security.agent.md`: add memory output, add/replace memory write step, update Anti-Drift
3. Read `ct-scalability.agent.md` same sections
4. Apply identical pattern to `ct-scalability.agent.md` with appropriate domain name
5. Read `ct-maintainability.agent.md` same sections
6. Apply identical pattern to `ct-maintainability.agent.md` with appropriate domain name
7. Verify all 3 files have consistent changes and no aggregator references

## Completion Checklist

- [x] ct-security.agent.md: Outputs include `memory/ct-security.mem.md`
- [x] ct-security.agent.md: "Write Isolated Memory" step with CT taxonomy `Highest Severity`
- [x] ct-security.agent.md: Anti-Drift references isolated memory
- [x] ct-scalability.agent.md: Outputs include `memory/ct-scalability.mem.md`
- [x] ct-scalability.agent.md: "Write Isolated Memory" step with CT taxonomy `Highest Severity`
- [x] ct-scalability.agent.md: Anti-Drift references isolated memory
- [ ] ct-maintainability.agent.md: Outputs include `memory/ct-maintainability.mem.md`
- [ ] ct-maintainability.agent.md: "Write Isolated Memory" step with CT taxonomy `Highest Severity`
- [ ] ct-maintainability.agent.md: Anti-Drift references isolated memory
- [ ] No aggregator references in any of the 3 files
- [ ] Section ordering preserved in all 3 files
- [ ] Completion contracts unchanged
