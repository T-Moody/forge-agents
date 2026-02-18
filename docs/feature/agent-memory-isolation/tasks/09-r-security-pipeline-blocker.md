# Task 09: R-Security — Pipeline Blocker Updates, Isolated Memory

## Task Goal

Update `r-security.agent.md` with targeted changes beyond the generic sub-agent pattern: rewrite the Pipeline Blocker Override Rule to reference the orchestrator (not R Aggregator), constrain severity vocabulary to Blocker/Major/Minor, emphasize the `Highest Severity` memory field as the sole vehicle for pipeline-blocking communication, and add isolated memory output.

## depends_on

none

## agent

implementer

## In-Scope

- **Outputs**: Add `docs/feature/<feature-slug>/memory/r-security.mem.md`
- **Pipeline Blocker Override Rule section** (current lines ~61–67): Rewrite to replace "the R Aggregator MUST override its aggregated result to `ERROR`" with "the orchestrator reads the `Highest Severity` field in `memory/r-security.mem.md` and declares pipeline `ERROR` if `Blocker` is found." Emphasize that the `Highest Severity` memory field is the sole vehicle for communicating blocker status
- **Severity vocabulary constraint**: Add explicit instruction: "In `Highest Severity`, use the R cluster canonical taxonomy: `Blocker/Major/Minor`. Do not use `Critical` — use `Blocker` instead."
- **Completion contract guidance** (current line ~224): Replace "the R Aggregator will override the pipeline result to ERROR" with "the orchestrator will read your `Highest Severity` memory field and declare pipeline ERROR if `Blocker` is found. You MUST populate `Highest Severity` with `Blocker` (not `Critical`) when any finding has pipeline-blocking severity."
- **Workflow**: Replace "No Memory Write" step with "Write Isolated Memory" step. The memory template for R-Security MUST include `Highest Severity` with explicit Blocker/Major/Minor value
- **Anti-Drift Anchor** (current line ~229): Keep "Your Critical/Blocker findings block the entire pipeline" but add: "via the `Highest Severity` field in your isolated memory file." Replace any R Aggregator references. Update memory write reference to isolated memory
- Remove all R Aggregator references throughout the file

## Out-of-Scope

- Modifying other R sub-agent files (tasks 13, 14 handle r-quality, r-testing, r-knowledge)
- Orchestrator R decision logic (task 03)
- R-Security's actual analysis methodology (unchanged)

## Acceptance Criteria

- AC-5: Outputs list `memory/r-security.mem.md`; workflow has "Write Isolated Memory" step
- AC-9 (partial): Pipeline Blocker Override Rule references orchestrator and `Highest Severity` memory field
- AC-16: Anti-Drift Anchor references isolated memory, no R Aggregator references
- AC-19: Section ordering preserved
- AC-20: Completion contract format unchanged (DONE/NEEDS_REVISION/ERROR)
- Severity vocabulary explicitly constrained to Blocker/Major/Minor (not "Critical")
- No "R Aggregator" or "R-Aggregator" references remain
- "No Memory Write" step replaced with "Write Isolated Memory"

## Estimated Effort

Medium

## Test Requirements

TDD skipped: markdown-only task, no behavioral code.

## Implementation Steps

1. Read `r-security.agent.md` fully to identify Pipeline Blocker Override Rule (~lines 61–67), No Memory Write step (~line 152), completion contract guidance (~line 224), Anti-Drift Anchor (~line 229), and all R Aggregator references
2. **Outputs**: Add `docs/feature/<feature-slug>/memory/r-security.mem.md (isolated memory)`
3. **Pipeline Blocker Override Rule** (lines ~61–67): Rewrite per design.md Revision 1:
   - Replace R Aggregator override language with orchestrator + memory field language
   - Add severity vocabulary constraint: "Use `Blocker/Major/Minor`. Do not use `Critical` — use `Blocker` instead."
   - Emphasize: "The `Highest Severity` field in your isolated memory file is the sole vehicle for communicating blocker status to the orchestrator"
4. **"No Memory Write" step** (line ~152): Replace with "Write Isolated Memory" step:
   ```
   ### N. Write Isolated Memory
   Write key findings to `memory/r-security.mem.md`:
   - Status: DONE/NEEDS_REVISION/ERROR with one-line summary
   - Key Findings: ≤5 bullet points of security findings
   - Highest Severity: Blocker/Major/Minor (use Blocker, NOT Critical, for pipeline-blocking findings)
   - Decisions Made: (omit if none)
   - Artifact Index: review/r-security.md — §Section pointers with brief relevance notes
   ```
5. **Completion contract guidance** (line ~224): Replace R Aggregator reference with orchestrator + memory field per design.md
6. **Anti-Drift Anchor** (line ~229): Add "via the `Highest Severity` field in your isolated memory file." Replace R Aggregator references. Add "You write only to your isolated memory file (`memory/r-security.mem.md`), never to shared `memory.md`"
7. Search for all remaining "R Aggregator", "R-Aggregator", "r-aggregator" references and remove/replace
8. Verify no aggregator references remain; verify severity vocabulary constraint is present

## Completion Checklist

- [x] Outputs include `memory/r-security.mem.md`
- [x] Pipeline Blocker Override Rule references orchestrator + `Highest Severity` memory field
- [x] Severity vocabulary constrained to Blocker/Major/Minor explicitly
- [x] "No Memory Write" step replaced with "Write Isolated Memory" including R-specific `Highest Severity`
- [x] Completion contract guidance references orchestrator (not R Aggregator)
- [x] Anti-Drift Anchor references isolated memory + blocker via memory field
- [x] No "R Aggregator" / "R-Aggregator" / "r-aggregator" references remain
- [x] Section ordering preserved
- [x] Completion contract format unchanged (DONE/NEEDS_REVISION/ERROR)

## Status: COMPLETE

TDD skipped: markdown-only task, no behavioral code.
