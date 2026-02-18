# Task 03: Orchestrator — Cluster Decision Logic (CT, V, R)

## Task Goal

Add the CT, V, and R cluster decision flows to the orchestrator, replacing the aggregator invocation steps. These compact decision tables enable the orchestrator to evaluate cluster results directly from isolated memory files.

## depends_on

02

## agent

implementer

## In-Scope

- **CT Cluster Decision Flow**: Add inline decision logic for reading CT memories, checking severity, determining DONE/NEEDS_REVISION/ERROR. Include self-verification logging step. Place this near the existing Step 3b section or as a referenced subsection.
- **V Cluster Decision Flow**: Add the V completion decision table (10 rows) for reading V memories, extracting statuses, applying the decision matrix. Include self-verification logging step. Place near Step 6 section.
- **R Cluster Decision Flow**: Add the R completion logic with R-Security priority override, R-Knowledge non-blocking rule, severity-based routing. Include self-verification logging step and severity mapping note ("Critical" → treat as "Blocker" for safety). Place near Step 7 section.
- **Dispatch pattern references**: Update the inline Pattern A/B/C references to remove "invoke aggregator" / "check aggregator completion contract" and replace with "orchestrator reads memories and determines cluster result".

## Out-of-Scope

- Full Workflow Steps rewrite (tasks 04 and 05 handle updating step-by-step instructions)
- NEEDS_REVISION routing table updates (task 05)
- Orchestrator Expectations table (task 05)
- Global Rules (already done in task 02)

## Acceptance Criteria

- AC-7: CT decision logic matches FR-4.1 — reads CT memories, Critical/High → NEEDS_REVISION, <2 outputs → ERROR, else DONE
- AC-8 (partial): V decision table matches FR-4.2 — 10-row matrix with mixed-state handling
- AC-9 (partial): R decision logic matches FR-4.3 — R-Security first, Blocker → ERROR, R-Knowledge non-blocking, <2 non-knowledge → ERROR, Major → NEEDS_REVISION
- Each decision flow includes a self-verification logging instruction
- Pattern A/B/C inline references updated to remove aggregator invocation
- No aggregator agent names appear in any decision flow

## Estimated Effort

Medium

## Test Requirements

TDD skipped: markdown-only task, no behavioral code.

## Implementation Steps

1. Read `orchestrator.agent.md` to locate Pattern A/B/C references (around lines 89–100) and Steps 3b, 6, 7
2. Update **Pattern A reference** (line ~89): remove "≥2 outputs required for aggregator" → "≥2 outputs required; orchestrator reads memories and determines cluster result"
3. Update **Pattern B reference** (line ~90): remove "forward to aggregator" → "orchestrator reads memories and determines cluster result"
4. Update **Pattern C reference** (lines ~92–100): replace aggregator references with orchestrator evaluation; keep max 3 iterations
5. Add **CT Cluster Decision Flow** subsection near Step 3b:
   ```
   CT Cluster Decision:
   1. Read memory/ct-security.mem.md, memory/ct-scalability.mem.md, memory/ct-maintainability.mem.md, memory/ct-strategy.mem.md
   2. Count available memories (agent returned without ERROR and file exists). <2 → cluster ERROR.
   3. Read "Highest Severity" from each available memory.
   4. Any severity == Critical OR High → NEEDS_REVISION (route to designer).
   5. Else → DONE (proceed to Step 4).
   Self-verification: Log severity values and routing decision in memory.md Recent Updates.
   ```
6. Add **V Cluster Decision Flow** subsection near Step 6:
   - Include the full 10-row decision table from design.md
   - Include self-verification logging
7. Add **R Cluster Decision Flow** subsection near Step 7:
   ```
   R Cluster Decision:
   1. Read memory/r-security.mem.md FIRST (mandatory).
   2. Missing or ERROR → pipeline ERROR.
   3. Highest Severity == Blocker → pipeline ERROR. (If "Critical" found, treat as "Blocker" for safety — flag prompt compliance gap.)
   4. Read remaining R memories (r-quality, r-testing, r-knowledge).
   5. R-Knowledge is NON-BLOCKING — ignore its ERROR.
   6. <2 non-knowledge memories available → ERROR.
   7. Any non-knowledge Highest Severity == Major → NEEDS_REVISION.
   8. Else → DONE.
   Self-verification: Log R memory statuses, severity values, and routing decision in memory.md Recent Updates.
   ```
8. Verify all 3 decision flows are present and match design.md specifications

## Status

COMPLETED

## Completion Checklist

- [x] CT decision flow present with severity check, ≥2 threshold, self-verification
- [x] V decision table present with all 9 rows (design.md specifies 9 rows, not 10), self-verification
- [x] R decision flow present with R-Security override first, R-Knowledge non-blocking, self-verification
- [x] R decision flow includes severity mapping note (Critical → Blocker)
- [x] Pattern A/B/C references updated to remove aggregator invocation (done in task 02)
- [x] No aggregator agent names in any decision flow or pattern reference
- [x] Standard section ordering preserved (Cluster Dispatch Patterns → Cluster Decision Logic → Workflow Steps)

## Implementation Notes

- TDD skipped: markdown-only task, no behavioral code.
- V decision table has 9 data rows matching both design.md and feature.md FR-4.2 (task description said "10 rows" but both source documents specify 9).
- Pattern A/B/C references were already updated by task 02; verified they contain no aggregator invocation language.
- Input Validation subsection added covering existence checks, format validation, and warning logging for all clusters.
- File grew from 398 to 461 lines (+63 lines for the Cluster Decision Logic section).
