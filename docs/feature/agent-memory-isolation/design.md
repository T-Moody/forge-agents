# Design: Agent-Isolated Memory System

**Summary:** Replace the shared `memory.md` write model and aggregator agents with a branch-merge memory architecture where each subagent writes a compact isolated memory file (`memory/<agent-name>.mem.md`), the orchestrator merges these into shared `memory.md`, and the orchestrator absorbs all aggregator decision logic. Three aggregator agents and the researcher synthesis mode are removed.

**Design Goals:**

1. Eliminate aggregator pipeline bottlenecks (4 sequential merge agents removed).
2. Preserve all safety-critical routing logic (R-Security override, CT severity gating, V decision table).
3. Keep the orchestrator concise despite absorbing new responsibilities (compact decision tables, not prose).
4. Maintain the non-blocking, fault-tolerant memory model.

---

## Context & Inputs

- [initial-request.md](initial-request.md) — Original request defining the branch-merge model, aggregator removal, and artifact changes.
- [feature.md](feature.md) — Full functional/non-functional requirements (FR-1 through FR-15, NFR-1 through NFR-6, AC-1 through AC-20).
- [research/architecture.md](research/architecture.md) — Repository structure, pipeline flow, aggregator architecture, memory system.
- [research/impact.md](research/impact.md) — Per-file impact analysis (tiers 1–5), cascade effects, risk areas.
- [research/dependencies.md](research/dependencies.md) — Data flow maps, input/output contracts, completion contract migration.
- [research/patterns.md](research/patterns.md) — Agent file conventions, memory access patterns, dispatch patterns, reusable patterns.

---

## High-Level Architecture

### Before (Current)

```
Sub-agents (parallel, read-only) → Aggregator (sequential, read-write) → Combined artifact → Downstream agent
                                         ↓
                                    memory.md write
```

### After (Proposed)

```
Sub-agents (parallel) → isolated memory files (memory/<agent>.mem.md) + primary artifacts
                                         ↓
             Orchestrator reads isolated memories → makes routing decision → merges into memory.md
                                         ↓
          Downstream agent reads upstream memories → consults artifact index → targeted artifact reads
```

### Component Responsibilities

| Component                                       | Current Responsibility                                                                                          | New Responsibility                                                                                                  |
| ----------------------------------------------- | --------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| **Orchestrator**                                | Dispatch agents, manage memory lifecycle (init/prune/invalidate), read aggregator completion contracts          | All current + read isolated memories + merge into `memory.md` + evaluate cluster completion (CT/V/R decision logic) |
| **Aggregator agents** (×3)                      | Read sub-agent artifacts, merge/dedup, produce combined artifact, write `memory.md`, return completion contract | **Removed**                                                                                                         |
| **Researcher synthesis mode**                   | Merge 4 research files into `analysis.md`, write `memory.md`                                                    | **Removed**                                                                                                         |
| **Sub-agents** (all)                            | Produce primary artifact; read `memory.md` (read-only)                                                          | All current + produce isolated memory file (`memory/<agent-name>.mem.md`)                                           |
| **Sequential agents** (spec, designer, planner) | Produce artifact; write to `memory.md` directly                                                                 | Produce artifact + isolated memory file; orchestrator merges memory (uniform model)                                 |

### Boundary Decisions

- **Orchestrator is sole `memory.md` writer.** All agents — including sequential agents that previously wrote directly — now write only to their isolated memory file. The orchestrator merges.
- **Orchestrator reads memories, not artifacts, for routing.** Cluster completion decisions use only the compact `*.mem.md` files. Downstream agents also navigate via memories first (see Memory-First Reading Pattern below).
- **No combined artifact files.** `analysis.md`, `design_critical_review.md`, `verifier.md`, `review.md` are eliminated. Downstream agents read upstream agent memories first, then use the artifact index within those memories to perform targeted reads of specific artifact sections.
- **Memory-first reading for all downstream agents.** No downstream agent reads a full upstream artifact file by default. Agents read the compact memory files (which serve as an index/summary), then selectively read only the artifact sections referenced by the memory's artifact index. This is analogous to reading a git log before selectively running `git show` on relevant commits.

---

## Data Models & DTOs

### Isolated Memory File Format

**Path convention:** `docs/feature/<feature-slug>/memory/<agent-name>.mem.md`

**Naming examples:**

| Agent                           | Memory File Path                         |
| ------------------------------- | ---------------------------------------- |
| researcher (architecture focus) | `memory/researcher-architecture.mem.md`  |
| researcher (impact focus)       | `memory/researcher-impact.mem.md`        |
| ct-security                     | `memory/ct-security.mem.md`              |
| v-build                         | `memory/v-build.mem.md`                  |
| r-security                      | `memory/r-security.mem.md`               |
| spec                            | `memory/spec.mem.md`                     |
| designer                        | `memory/designer.mem.md`                 |
| planner                         | `memory/planner.mem.md`                  |
| implementer (task T01)          | `memory/implementer-T01.mem.md`          |
| documentation-writer (task T05) | `memory/documentation-writer-T05.mem.md` |

**Target size:** ≤30 lines (NFR-2).

**Template (all agents):**

```markdown
# Memory: <agent-name>

## Status

<DONE|NEEDS_REVISION|ERROR>: <one-line summary>

## Key Findings

- <finding 1>
- <finding 2>
- ... (≤5 bullets)

## Highest Severity

<severity-value or N/A>

<!-- CT agents: Critical/High/Medium/Low -->
<!-- R agents: Blocker/Major/Minor. R-Security MUST use "Blocker" (not "Critical") to align with the R cluster canonical taxonomy. -->
<!-- V agents: PASS/FAIL -->
<!-- Others: N/A -->

## Decisions Made

- <decision 1> (≤2 sentences)
<!-- Omit section if no decisions -->

## Artifact Index

- <file-path-1> — §<Section> (brief relevance note), §<Section> (brief relevance note)
- <file-path-2> — §<Section> (brief relevance note)
<!-- Section pointers (§) tell downstream agents which sections to read. -->
```

**Cluster-specific fields:**

- **CT sub-agents** MUST populate `Highest Severity` with their highest-severity finding (Critical/High/Medium/Low). Required for FR-4.1/FR-4.5.
- **R sub-agents** MUST populate `Highest Severity` with their highest-severity finding using the canonical R taxonomy: **Blocker/Major/Minor**. R-Security's existing prompt uses both "Blocker" and "Critical" interchangeably; the memory template constrains this to **"Blocker"** only ("Critical" is not a valid R taxonomy value). R-Security MUST include explicit severity for FR-4.3/FR-4.6.
- **V sub-agents** MUST populate `Highest Severity` with PASS or FAIL. V-Tests, V-Tasks, V-Feature additionally include their completion status (DONE/NEEDS_REVISION/ERROR) in the Status line.
- **V-Build** uses PASS/FAIL in `Highest Severity` (binary gate).

### Updated `memory.md` Template

No structural change to the shared `memory.md` template. It retains its four sections:

```markdown
# Operational Memory

## Artifact Index

| Artifact | Key Sections | Last Updated By |

## Recent Decisions

<!-- Format: - [agent-name, step-N] Decision summary. Rationale: ... -->

## Lessons Learned

<!-- Never pruned. Format: - [agent-name, step-N] Issue → Resolution. -->

## Recent Updates

<!-- Format: - [agent-name, step-N] Updated `artifact-path` — summary. -->
```

The orchestrator populates this by reading isolated memory files and extracting:

- `Artifact Index` entries (with section pointers) → Artifact Index rows (preserving section-level granularity so downstream agents can navigate to specific sections)
- `Decisions Made` → Recent Decisions entries
- `Key Findings` summary → Recent Updates entries

### Updated Documentation Structure

Files under `docs/feature/<feature-slug>/`:

| Path                              | Created By                 | Purpose                                  |
| --------------------------------- | -------------------------- | ---------------------------------------- |
| `initial-request.md`              | Orchestrator               | User's original request                  |
| `memory.md`                       | Orchestrator (sole writer) | Shared operational memory                |
| `memory/`                         | All agents                 | Directory for isolated memory files      |
| `memory/<agent-name>.mem.md`      | Each agent                 | Compact memory for orchestrator routing  |
| `research/*.md`                   | Researcher (focused ×4)    | Individual research outputs              |
| `feature.md`                      | Spec                       | Feature specification                    |
| `design.md`                       | Designer                   | Technical design                         |
| `ct-review/ct-*.md`               | CT sub-agents ×4           | Individual CT reviews                    |
| `plan.md`                         | Planner                    | Implementation plan                      |
| `tasks/*.md`                      | Planner                    | Individual task files                    |
| `verification/v-*.md`             | V sub-agents ×4            | Individual verification reports          |
| `review/r-*.md`                   | R sub-agents ×4            | Individual review reports                |
| `review/knowledge-suggestions.md` | R-Knowledge                | Knowledge evolution proposals            |
| `decisions.md`                    | R-Knowledge                | Cross-feature architectural decision log |

**Removed:** `analysis.md`, `design_critical_review.md`, `verifier.md`, `review.md`.

---

## APIs & Interfaces

### Agent Prompt Contract Changes

Every agent file follows the convention: Inputs → Outputs → Operating Rules → Workflow → Completion Contract → Anti-Drift Anchor. The changes below define how each section is modified.

#### Universal Changes (all active agents except deprecated `critical-thinker`)

**Outputs section — add:**

```markdown
- docs/feature/<feature-slug>/memory/<agent-name>.mem.md (isolated memory)
```

**Workflow — add final step before completion contract:**

```markdown
### N. Write Isolated Memory

Write key findings to `memory/<agent-name>.mem.md`:

- Status: completion status (DONE/ERROR)
- Key Findings: ≤5 bullet points summarizing primary findings
- Highest Severity: highest severity rating found (or N/A)
- Decisions Made: any decisions taken (omit if none)
- Artifact Index: list of output file paths with section-level pointers (§Section Name) and brief relevance notes for each key section
```

**Anti-Drift Anchor — replace memory-write prohibition:**

- Old: "You do NOT write to `memory.md`" / "You never write to `memory.md`"
- New: "You write only to your isolated memory file (`memory/<agent-name>.mem.md`), never to shared `memory.md`."

**"No Memory Write" workflow step — replace (V and R sub-agents):**

- Old: "(No memory write step — findings are communicated through `<artifact>`. The <Aggregator> will consolidate relevant findings into memory after all sub-agents complete.)"
- New: The "Write Isolated Memory" step above.

#### Orchestrator-Specific Changes

**Outputs section — update:**

```markdown
- docs/feature/<feature-slug>/initial-request.md
- docs/feature/<feature-slug>/memory.md (sole writer — lifecycle management + merge)
- docs/feature/<feature-slug>/memory/ (directory — created at Step 0)
- Coordination of all subagent invocations
```

**Global Rule 6 — Memory-First Protocol — update:**

- Add "Merge" action: after each agent or cluster completes, read isolated memory file(s) and merge into `memory.md`.
- Remove pruning checkpoint at "Step 1.2" (Step 1.2 no longer exists). Prune after Steps 1.1 (post-research merge), 2, 4.

**Global Rule 10 — Approval Gate — update:**

- Replace "after Step 1.2 (research synthesis)" with "after Step 1.1 (research completion)".

**Global Rule 12 — Memory Write Safety — update:**

```markdown
12. **Memory Write Safety:** The orchestrator is the sole writer to shared `memory.md`.
    All other agents write only to their own isolated memory file (`memory/<agent-name>.mem.md`).
    - **Isolated memory (all agents):** Each agent writes to `memory/<agent-name>.mem.md` only.
    - **Shared memory (orchestrator only):** The orchestrator reads isolated memories and merges into `memory.md` after each cluster/agent completes.
```

Remove the read-only/read-write lists entirely — the distinction is no longer needed since no agent writes to shared `memory.md`.

#### Downstream Input Changes

Downstream agents follow the **memory-first reading pattern**: read upstream agent memories first, consult the artifact index within those memories, then perform targeted reads of only the artifact sections indicated by the index. No agent reads a full artifact file unless the memory index directs it to.

**`spec.agent.md` — Inputs:**

- Remove: `analysis.md`
- Add (primary): `memory/researcher-architecture.mem.md`, `memory/researcher-impact.mem.md`, `memory/researcher-dependencies.mem.md`, `memory/researcher-patterns.mem.md` — read these first for orientation and artifact index
- Add (selective): `research/architecture.md`, `research/impact.md`, `research/dependencies.md`, `research/patterns.md` — read only the sections referenced in the researcher memory artifact indexes

**Reading workflow:** Spec reads the 4 researcher memories → reviews key findings and artifact indexes → identifies which sections of each `research/*.md` file are relevant to feature specification → reads only those sections.

**`designer.agent.md` — Inputs:**

- Remove: `analysis.md`
- Remove: `design_critical_review.md (if exists — read during revision cycle)`
- Add (primary): `memory/spec.mem.md`, `memory/researcher-*.mem.md` — read these first for orientation and artifact index
- Add (selective): `research/architecture.md`, `research/impact.md`, `research/dependencies.md`, `research/patterns.md` — read only sections referenced in researcher memory artifact indexes
- Add (selective, revision mode): `memory/ct-*.mem.md` (if exists) — read CT memories first → then selectively read `ct-review/ct-security.md`, `ct-review/ct-scalability.md`, `ct-review/ct-maintainability.md`, `ct-review/ct-strategy.md` sections based on CT memory artifact indexes

**Reading workflow:** Designer reads spec memory + researcher memories → reviews key findings and artifact indexes → identifies relevant sections of `feature.md` and `research/*.md` → reads only those sections. In revision mode: reads CT memories → identifies which CT findings require design changes → reads only the relevant sections of individual `ct-review/ct-*.md` files.

**`planner.agent.md` — Inputs:**

- Remove: `analysis.md`
- Remove: `design_critical_review.md (if present)`
- Remove: `verifier.md (replan mode)`
- Add (primary): `memory/designer.mem.md`, `memory/spec.mem.md` — read these first for orientation and artifact index
- Add (selective): `design.md`, `feature.md` — read only sections referenced in designer/spec memory artifact indexes
- Add (selective): `research/architecture.md`, `research/impact.md`, `research/dependencies.md`, `research/patterns.md` — read only sections referenced in researcher memory artifact indexes (via `memory.md` Artifact Index)
- Add (selective, if present): `memory/ct-*.mem.md` → selectively read `ct-review/ct-*.md` sections (planning constraints from CT review findings)
- Add (replan mode, primary): `memory/v-build.mem.md`, `memory/v-tests.mem.md`, `memory/v-tasks.mem.md`, `memory/v-feature.mem.md` — read these first → selectively read `verification/v-tasks.md`, `verification/v-tests.md`, `verification/v-feature.md` sections based on V memory artifact indexes

**Reading workflow:** Planner reads designer memory + spec memory → reviews key findings and artifact indexes → identifies relevant sections of `design.md` and `feature.md` → reads those sections. In replan mode: reads V sub-agent memories → identifies failing tasks/tests/criteria from memory key findings → reads only the relevant failure sections of `verification/v-*.md` files.

**`implementer.agent.md` — Inputs:**

- Add (primary): `memory/planner.mem.md` — read first for orientation and task-relevant artifact index
- Add (selective): task file (`tasks/<task-id>.md`) + relevant sections of `design.md` referenced in planner memory artifact index

**Reading workflow:** Implementer reads planner memory → identifies its assigned task and relevant design sections from the artifact index → reads its task file and only the referenced design sections.

**`planner.agent.md` — Replan Mode Detection (replaces `verifier.md` existence check):**

The current planner detects replan mode via `verifier.md` existence (line 59: "Replan mode: `verifier.md` exists with failures"). Since `verifier.md` is removed, the design defines a two-part replacement mechanism:

1. **Orchestrator explicit mode signal (primary):** The orchestrator dispatches the planner in replan mode by including the string `MODE: REPLAN` and the paths to V memory files (`memory/v-*.mem.md`) in the dispatch prompt. The planner's mode detection logic is updated to: "Replan mode: orchestrator provides MODE: REPLAN signal with V memory paths."
2. **V artifact existence check (redundant safety):** As a backup, the planner also checks: if `verification/v-tasks.md` exists AND contains failing task IDs (non-empty `## Failures` or equivalent section), treat as replan mode even without explicit orchestrator signal.

The orchestrator's Step 6.4 (Pattern C replan) is updated to include the explicit mode signal when invoking the planner: "Invoke planner with MODE: REPLAN and paths: `memory/v-build.mem.md`, `memory/v-tests.mem.md`, `memory/v-tasks.mem.md`, `memory/v-feature.mem.md`."

**`planner.agent.md` — Replan Cross-Referencing Workflow (new):**

The planner's replan mode workflow MUST include explicit cross-referencing steps to replace the structured Actionable Items previously provided by v-aggregator. The planner first reads V sub-agent memories to orient and identify which artifact sections contain failures (memory-first reading), then performs targeted reads for cross-referencing. Add the following workflow steps to the planner's replan procedure:

```
Replan Mode Cross-Referencing Steps:
0. Read V sub-agent memories (memory/v-*.mem.md) → review key findings and
   artifact indexes → identify which sections of each verification artifact
   contain failures. Use artifact index section pointers for targeted reads below.
1. Read targeted sections of verification/v-tasks.md (per artifact index) →
   Extract all failing task IDs and their failure reasons.
   Build a list: { TaskID → Failure Reason }
2. Read targeted sections of verification/v-tests.md (per artifact index) →
   Identify test failures.
   For each failing test, match to a task ID by examining the test's file path,
   task reference, or the code under test. Append to task failure list:
   { TaskID → [Failure Reason, Related Test Failures] }
3. Read targeted sections of verification/v-feature.md (per artifact index) →
   Identify unmet acceptance criteria.
   For each unmet criterion, identify the responsible task ID(s) from plan.md.
   Append to task failure list:
   { TaskID → [Failure Reason, Related Test Failures, Unmet Criteria] }
4. Prioritize tasks for remediation:
   - Tasks with both test failures AND unmet criteria → highest priority
   - Tasks with test failures only → medium priority
   - Tasks with unmet criteria only → lower priority
5. Produce remediation plan targeting the prioritized task list.
   Do NOT re-plan already-completed tasks that are not in the failure list.
```

**`researcher.agent.md`:**

- Remove: Mode 2 (Synthesis) entirely — all synthesis mode inputs, outputs, workflow, completion contract
- Remove: `analysis.md` from outputs
- Keep: Focused mode only

---

## Sequence / Interaction Notes

### Updated Pipeline Flow

```
Step 0:   Orchestrator → initial-request.md, memory.md, memory/ directory
Step 1.1: Researcher ×4 (parallel, Pattern A) → research/*.md + memory/researcher-*.mem.md
Step 1.1m: Orchestrator reads 4 researcher memories → merge to memory.md → prune
Step 1.1a: (Conditional) Approval Gate — Post-Research
Step 2:   Spec reads researcher memories → consults artifact indexes → targeted reads of research/*.md sections → feature.md + memory/spec.mem.md
Step 2m:  Orchestrator merges spec memory → prune
Step 3:   Designer reads spec + researcher memories → consults artifact indexes → targeted reads of feature.md + research/*.md sections → design.md + memory/designer.mem.md
Step 3m:  Orchestrator merges designer memory
Step 3b:  CT ×4 (parallel, Pattern A) → ct-review/ct-*.md + memory/ct-*.mem.md
Step 3bm: Orchestrator reads 4 CT memories → evaluate severity → merge to memory.md
Step 3b.r: (Conditional) NEEDS_REVISION → designer reads CT memories → consults artifact indexes → targeted reads of ct-review/ct-*.md sections → revises design.md → re-run CT (max 1 loop). If still NEEDS_REVISION after loop: proceed with warning; forward all High/Critical findings from individual CT artifacts as planning constraints to planner (Step 4).
Step 4:   Planner reads designer + spec memories → consults artifact indexes → targeted reads of design.md, feature.md, research/*.md sections, [CT memories → ct-review/ct-*.md sections] → plan.md, tasks/*.md + memory/planner.mem.md
Step 4m:  Orchestrator merges planner memory → prune
Step 4a:  (Conditional) Approval Gate — Post-Planning
Step 5:   Implementers read planner memory → consult artifact index → targeted reads of task files + design.md sections → code/tests/docs + memory/implementer-*.mem.md
Step 5m:  Between waves: orchestrator reads implementer memories → merge + extract Lessons Learned
Step 6:   V cluster (Pattern B+C):
          6.1: V-Build → verification/v-build.md + memory/v-build.mem.md
          6.2: V-Tests, V-Tasks, V-Feature (parallel) → verification/v-*.md + memory/v-*.mem.md
          6.3m: Orchestrator reads 4 V memories → evaluate decision table → merge to memory.md
          6.4: (Conditional) NEEDS_REVISION → orchestrator invokes planner with MODE: REPLAN + V memory paths → planner reads V memories → consults artifact indexes → targeted reads of verification/v-*.md failure sections → cross-references failures → fix tasks → re-run V cluster (max 3 loops)
Step 7:   R ×4 (parallel, Pattern A) → review/r-*.md + memory/r-*.mem.md
Step 7m:  Orchestrator reads 4 R memories → evaluate R logic (security override) → merge to memory.md
Step 7.r: Handle R result (DONE / NEEDS_REVISION / ERROR per decision logic)
Step 7.5: Preserve knowledge-suggestions.md and decisions.md
```

### Orchestrator Memory Merge Sequence (per cluster)

```
1. All sub-agents in cluster return (DONE/ERROR)
2. Orchestrator reads each memory/<agent>.mem.md
3. For each memory file:
   a. Extract Artifact Index entries (with section pointers) → append to memory.md Artifact Index (preserving section-level granularity)
   b. Extract Decisions Made → append to memory.md Recent Decisions
   c. Extract Key Findings summary → append to memory.md Recent Updates
4. Apply cluster completion logic (CT/V/R specific — see below)
5. Continue pipeline or route NEEDS_REVISION
```

### CT Cluster Decision Flow

```
1. Read memory/ct-security.mem.md, memory/ct-scalability.mem.md,
   memory/ct-maintainability.mem.md, memory/ct-strategy.mem.md
2. Count available memories (agent returned without ERROR and memory file exists)
3. If <2 memories available → cluster ERROR
4. Read "Highest Severity" from each available memory
5. If any severity == Critical OR High → NEEDS_REVISION (route to designer)
6. Else → DONE (proceed to Step 4)

Self-verification: Log the severity values found across CT memories and the
resulting routing decision in memory.md Recent Updates before proceeding.
```

### V Cluster Decision Flow

```
1. Read memory/v-build.mem.md, memory/v-tests.mem.md,
   memory/v-tasks.mem.md, memory/v-feature.mem.md
2. Extract Status from each memory
3. Apply decision table:

   | V-Build | V-Tests | V-Tasks | V-Feature | Result |
   |---------|---------|---------|-----------|--------|
   | DONE    | DONE    | DONE    | DONE      | DONE   |
   | DONE    | NR      | any     | any       | NR     |
   | DONE    | any     | NR      | any       | NR     |
   | DONE    | any     | any     | NR        | NR     |
   | ERROR   | any     | any     | any       | ERROR  |
   | DONE    | ERROR   | !ERROR  | !ERROR    | Proceed|
   | DONE    | !ERROR  | ERROR   | !ERROR    | Proceed|
   | DONE    | !ERROR  | !ERROR  | ERROR     | Proceed|
   | any     | —       | —       | —         | 2+ missing/ERROR → ERROR |

   NR = NEEDS_REVISION; Proceed = proceed with available
4. If NEEDS_REVISION or ERROR → Pattern C replan loop

Self-verification: Log the V sub-agent statuses and the resulting cluster
decision in memory.md Recent Updates before proceeding.
```

### R Cluster Decision Flow

```
1. Read memory/r-security.mem.md FIRST (mandatory, pipeline-critical)
2. If r-security memory missing or Status == ERROR → pipeline ERROR
3. If r-security Highest Severity == Blocker → pipeline ERROR
   (Note: R-Security's memory template constrains this field to Blocker/Major/Minor.
    If "Critical" is found, treat as "Blocker" for safety — but this indicates a
    prompt compliance gap that should be flagged for correction.)
4. Read memory/r-quality.mem.md, memory/r-testing.mem.md, memory/r-knowledge.mem.md
5. R-Knowledge status is NON-BLOCKING — ignore ERROR from r-knowledge
6. Count available non-knowledge memories (r-security, r-quality, r-testing)
7. If <2 non-knowledge memories available → ERROR
8. If any non-knowledge memory Highest Severity == Major → NEEDS_REVISION
9. Else → DONE

Self-verification: Before proceeding, the orchestrator SHOULD log which R memories
were read, what severity values were found, and the resulting routing decision in
the memory.md Recent Updates section. This provides auditability for the most
safety-critical routing decision in the pipeline.
```

---

## Security Considerations

### Authentication/Authorization

N/A — This is a prompt-file-only change. No authentication, API keys, or user credentials are involved. The system operates through LLM agent invocations with file-based I/O.

### Data Protection

- **No sensitive data in memory files.** Isolated memory files contain finding summaries, severity levels, and file paths. No secrets, credentials, or PII.
- **R-Security pipeline override is safety-critical.** The design preserves the existing R-Security override rule: R-Security Blocker findings block the pipeline (the `Highest Severity` memory field is the vehicle; see Revision 5 for taxonomy standardization). This logic moves from r-aggregator to the orchestrator (FR-4.3, FR-4.6). The override is checked first in the R decision flow (step 2-3 above).

### Threat Model

- **Risk: R-Security override bypassed.** If the orchestrator's R decision logic is implemented incorrectly, security-critical findings could be silently passed. **Mitigation:** The R decision flow explicitly checks R-Security first, before any other R sub-agent. AC-9 explicitly tests for this.
- **Risk: Orchestrator skips memory merge.** If merge is skipped, downstream agents lose navigation context. **Mitigation:** Memory is non-blocking (NFR-4); agents fall back to direct artifact reads.

### Input Validation Strategy

- **Memory file existence check:** The orchestrator validates that expected `memory/<agent>.mem.md` files exist after each cluster. Missing files are logged as warnings and the agent's findings are read from primary artifacts as fallback.
- **Severity field validation:** The orchestrator checks that `Highest Severity` values match the expected taxonomy (Critical/High/Medium/Low for CT; Blocker/Major/Minor for R; PASS/FAIL for V). Unexpected values are treated as the worst-case severity for safety.

---

## Failure & Recovery

### Expected Failure Modes

| Failure Mode                                               | Likelihood | Impact   | Recovery Strategy                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| ---------------------------------------------------------- | ---------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Agent fails to create isolated memory file**             | Low        | Medium   | Non-blocking. Orchestrator logs warning, reads primary artifact for routing if needed. Pipeline continues (NFR-4).                                                                                                                                                                                                                                                                                                                                |
| **`memory/` directory does not exist**                     | Low        | Medium   | Orchestrator creates `memory/` at Step 0 initialization. If creation fails at Step 0, agents create on first write. If all creation fails, fall back to non-blocking degradation.                                                                                                                                                                                                                                                                 |
| **Fewer than 2 sub-agent outputs in cluster**              | Low        | High     | Cluster ERROR. Same threshold as current aggregator system.                                                                                                                                                                                                                                                                                                                                                                                       |
| **R-Security agent fails or missing**                      | Low        | Critical | Pipeline ERROR. R-Security is mandatory. Retry once per Global Rule 4; if persistent, halt pipeline.                                                                                                                                                                                                                                                                                                                                              |
| **R-Knowledge agent fails**                                | Low        | Low      | Non-blocking. Orchestrator ignores R-Knowledge failure. Pipeline continues.                                                                                                                                                                                                                                                                                                                                                                       |
| **V-Build gate fails**                                     | Medium     | High     | Skip parallel V agents, declare V cluster ERROR. Pattern C replan loop triggers.                                                                                                                                                                                                                                                                                                                                                                  |
| **Pattern C exhausts 3 iterations**                        | Low        | Medium   | Proceed with findings in individual V sub-agent artifacts. Pipeline moves to Step 7 (Review).                                                                                                                                                                                                                                                                                                                                                     |
| **Memory merge conflict (contradictory findings)**         | Medium     | Medium   | No conflict resolution. Sequential append — both entries merged. Routing uses worst-case severity across all memories.                                                                                                                                                                                                                                                                                                                            |
| **Missing research file for downstream agent**             | Low        | Medium   | Downstream agent notes gap per Operating Rule 2 (missing context) and proceeds with available files.                                                                                                                                                                                                                                                                                                                                              |
| **Orchestrator memory merge fails**                        | Low        | Medium   | Non-blocking. Log warning. Downstream agents use direct artifact reads.                                                                                                                                                                                                                                                                                                                                                                           |
| **Planner replan without aggregated task-failure mapping** | Medium     | High     | Planner reads V sub-agent memories for orientation and artifact index, then performs targeted reads of `verification/v-tasks.md` for failing task IDs, `verification/v-tests.md` for test failures, and `verification/v-feature.md` for unmet criteria. Follows explicit cross-referencing workflow steps added to planner prompt (see APIs & Interfaces → planner replan cross-referencing). Self-assembles prioritized failure-to-task mapping. |

### Retry/Fallback Strategies

- **Agent ERROR:** Retry once per Global Rule 4 (existing behavior, unchanged).
- **Memory file missing:** Fall back to reading primary artifact. Non-blocking.
- **Severity field missing/invalid in isolated memory:** Treat as worst-case severity for the cluster's taxonomy (Critical for CT, Blocker for R, FAIL for V).
- **Memory merge failure:** Log warning, continue. Memory is beneficial but not required.

### Graceful Degradation

The entire isolated memory system is designed to degrade gracefully. If any memory operation fails:

1. Agents still produce their primary artifacts (unchanged).
2. The orchestrator can read primary artifacts directly as a fallback for routing decisions (less efficient but functional).
3. The pipeline continues — memory failures never halt the pipeline (except R-Security, which is a content failure, not a memory failure).

---

## Non-Functional Requirements

### NFR-1: Orchestrator Prompt Size

- The orchestrator absorbs 3 aggregators' decision logic. To keep the prompt manageable:
  - CT decision logic: ~8 lines (severity check + self-verification).
  - V decision logic: compact table (~15 lines including table + surrounding prose).
  - R decision logic: ~12 lines (ordered priority rules + self-verification + severity mapping note).
  - Memory merge protocol: ~5 lines per invocation point, ~8 merge step references across the workflow.
  - Updated NEEDS_REVISION routing table: modified rows (not shorter).
  - `memory/` directory creation at Step 0: ~2 lines.
  - **Revised estimate:** +50–80 lines for decision logic, merge steps, and updated references; −20 lines for removing aggregator invocation steps and combined artifact references. **Net growth ≈ +30–60 lines**, pushing the orchestrator from 399 to an estimated **~430–460 lines**.
  - **Complexity density note:** The line count understates the cognitive complexity. The three aggregator agents currently total ~750 lines of detailed procedural instructions. The orchestrator compresses their _routing decision logic_ into ~35 lines of compact tables — but those lines carry the cognitive weight of what was previously spread across substantive step-by-step procedures. The orchestrator's error surface for routing decisions increases accordingly.
- Decision tables are expressed as compact tables/lists, not prose. Each decision flow includes a self-verification step (log reasoning before proceeding) to mitigate LLM reasoning errors in compressed decision logic.
- **Extraction threshold (firm):** If the post-implementation orchestrator exceeds **450 lines**, the CT/V/R decision tables MUST be extracted to `dispatch-patterns.md` (or a new `cluster-decisions.md`) as a **blocking follow-up task** before the feature is considered complete. This is not a deferred contingency — it is a planned conditional step.
- Pattern C replan loop retains its inline definition but replaces `verifier.md` references with V memories.

### NFR-2: Memory File Compactness

- Target ≤30 lines per isolated memory file.
- The template above is 15–25 lines depending on content.
- Key Findings limited to ≤5 bullet points.

### NFR-3: Convention Consistency

- All agent files retain standard section ordering: YAML frontmatter → Title → Role → Prohibitions → Inputs → Outputs → Operating Rules → Workflow → Completion Contract → Anti-Drift Anchor.
- Operating Rules 1–4 remain verbatim across all agents.
- Operating Rule 6 (Memory-first reading) is preserved — agents still read `memory.md` first to orient.
- New isolated memory write step is added as the final workflow step (before completion contract).

### NFR-4: Non-Blocking Memory

- If an agent cannot create its isolated memory file, the pipeline continues.
- If the orchestrator cannot merge a memory, the pipeline continues.
- All fallback paths use direct artifact reads.

### NFR-5: Backward-Compatible Severity Taxonomies

- CT: Critical / High / Medium / Low (unchanged).
- R: **Blocker / Major / Minor** (canonical R taxonomy — unchanged in meaning). Note: R-Security's existing prompt uses "Critical" and "Blocker" interchangeably (line 66: `Severity: Blocker (Critical)`). Post-change, the memory template constrains R agents to use only **Blocker/Major/Minor**. The term "Critical" is NOT part of the R severity taxonomy; R-Security is explicitly instructed to use "Blocker" for pipeline-blocking findings. The orchestrator's R decision flow treats any unexpected severity value (including "Critical") as worst-case ("Blocker") for safety, but this fallback should not be relied upon — it indicates a prompt compliance gap.
- V: PASS / FAIL + task-level DONE/NEEDS_REVISION/ERROR (unchanged).

### NFR-6: Concurrency Safety

- Parallel agents write only to their own `memory/<agent-name>.mem.md` file — no shared-state conflicts.
- The orchestrator merges sequentially after all parallel agents return — no concurrent writes to `memory.md`.

---

## Migration & Backwards Compatibility

### Files to Remove (Deprecate)

| File                     | Action                                   | Deprecation Notice                                                                                                                                                                                              |
| ------------------------ | ---------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ct-aggregator.agent.md` | Replace contents with deprecation notice | "DEPRECATED: CT cluster decision logic has moved to the orchestrator. Individual CT sub-agents write isolated memory files. The orchestrator reads these memories and applies severity-based routing directly." |
| `v-aggregator.agent.md`  | Replace contents with deprecation notice | "DEPRECATED: V cluster decision logic has moved to the orchestrator. Individual V sub-agents write isolated memory files. The orchestrator reads these memories and applies the V decision table directly."     |
| `r-aggregator.agent.md`  | Replace contents with deprecation notice | "DEPRECATED: R cluster decision logic has moved to the orchestrator. Individual R sub-agents write isolated memory files. The orchestrator reads these memories and applies R completion logic directly."       |

### Removed Artifacts

| Artifact                    | Previously Produced By | Downstream Impact                                                                             |
| --------------------------- | ---------------------- | --------------------------------------------------------------------------------------------- |
| `analysis.md`               | researcher (synthesis) | Spec, designer, planner read researcher memories → targeted reads of `research/*.md` sections |
| `design_critical_review.md` | ct-aggregator          | Designer reads CT memories → targeted reads of `ct-review/ct-*.md` sections in revision mode  |
| `verifier.md`               | v-aggregator           | Planner reads V memories → targeted reads of `verification/v-*.md` sections in replan mode    |
| `review.md`                 | r-aggregator           | Orchestrator reads R memories directly                                                        |

### Reference Cleanup

All string references to the following must be removed from active files:

- Agent names: `ct-aggregator`, `v-aggregator`, `r-aggregator`, `CT Aggregator`, `V Aggregator`, `R Aggregator`, `CT-Aggregator`, `V-Aggregator`, `R-Aggregator`
- Artifact paths: `analysis.md` (as artifact reference), `design_critical_review.md`, `verifier.md` (as artifact path), `review.md` (as artifact produced by r-aggregator)
- Mode references: `synthesis mode`, `Mode 2`, `Synthesize`

### Researcher Synthesis Mode Removal

- Remove Mode 2 section from `researcher.agent.md` (synthesis workflow, analysis.md contents template, synthesis completion contract).
- Remove synthesis mode inputs (research/\*.md as merge inputs).
- Remove `analysis.md` from outputs.
- Retain only focused mode.

### Sequential Agent Memory Model Migration

Sequential agents (spec, designer, planner) previously wrote to `memory.md` directly. Under the new model:

- They write to their isolated memory file instead.
- The orchestrator merges their memory into `memory.md` after they return.
- This is functionally equivalent (since they run sequentially, the orchestrator can merge immediately) but maintains a uniform branch-merge model across all agents.
- Their workflow step "Update `memory.md`" is replaced with "Write Isolated Memory" (same content, different target file).

---

## Testing Strategy

Since all files are markdown prompt definitions, testing is structural verification of the modified files.

### Unit-Level Verification

| Test                                    | Verifies            | Method                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| --------------------------------------- | ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **T1: Aggregator removal**              | AC-1                | Verify `ct-aggregator.agent.md`, `v-aggregator.agent.md`, `r-aggregator.agent.md` contain only deprecation notices (no Inputs, Outputs, Operating Rules, Workflow, Completion Contract sections).                                                                                                                                                                                                                                                              |
| **T2: No aggregator references**        | AC-2                | Grep all active `.agent.md` and `.md` files in `NewAgentsAndPrompts/` for `ct-aggregator`, `v-aggregator`, `r-aggregator`, `CT Aggregator`, `V Aggregator`, `R Aggregator`, `CT-Aggregator`, `V-Aggregator`, `R-Aggregator`. Zero matches expected (excluding deprecation notices in aggregator files).                                                                                                                                                        |
| **T3: No removed artifact references**  | AC-3                | Grep all active files for `analysis.md`, `design_critical_review.md`, `verifier.md` (as artifact paths — not substring matches like "verifier" in generic text), `review.md` (as artifact path produced by r-aggregator). Zero matches in active files.                                                                                                                                                                                                        |
| **T4: No synthesis mode**               | AC-4, AC-18         | Verify `researcher.agent.md` contains no "Mode 2", "Synthesis", "synthesis mode", or `analysis.md`. Verify `orchestrator.agent.md` contains no Step 1.2 or synthesis invocation.                                                                                                                                                                                                                                                                               |
| **T5: Isolated memory in all agents**   | AC-5                | For each of 21 active agent files: verify Outputs section contains `memory/<agent-name>.mem.md`; verify Workflow contains a "Write Isolated Memory" step.                                                                                                                                                                                                                                                                                                      |
| **T6: Orchestrator memory merge**       | AC-6                | Verify `orchestrator.agent.md` contains: (a) "Merge" action in Memory Lifecycle, (b) merge steps after each cluster, (c) "sole writer to `memory.md`" statement.                                                                                                                                                                                                                                                                                               |
| **T7: CT decision logic**               | AC-7                | Verify orchestrator Step 3b contains: read CT memories → check severity → DONE/NEEDS_REVISION/ERROR logic matching FR-4.1.                                                                                                                                                                                                                                                                                                                                     |
| **T8: V decision logic**                | AC-8                | Verify orchestrator Step 6 contains: V decision table matching FR-4.2, Pattern C loop references individual V artifacts (not `verifier.md`).                                                                                                                                                                                                                                                                                                                   |
| **T9: R decision logic**                | AC-9                | Verify orchestrator Step 7 contains: R-Security override (first check), R-Knowledge non-blocking, severity-based routing matching FR-4.3.                                                                                                                                                                                                                                                                                                                      |
| **T10: Downstream inputs**              | AC-10, AC-11, AC-12 | Verify spec, designer, planner Inputs sections list upstream memory files as primary inputs (not `analysis.md` or direct artifact reads). Verify memory-first reading workflow is documented: read memories → consult artifact index → targeted artifact section reads. Designer lists CT memories for revision (not `design_critical_review.md`). Planner lists V memories for replan (not `verifier.md`). Implementer lists planner memory as primary input. |
| **T11: Dispatch patterns**              | AC-13               | Verify `dispatch-patterns.md` Pattern A and B contain no aggregator invocation steps or combined artifact references.                                                                                                                                                                                                                                                                                                                                          |
| **T12: Memory write safety**            | AC-14               | Verify Global Rule 12 states orchestrator is sole `memory.md` writer. No aggregators listed.                                                                                                                                                                                                                                                                                                                                                                   |
| **T13: Documentation structure**        | AC-15               | Verify Documentation Structure table includes `memory/` directory, excludes `analysis.md`, `design_critical_review.md`, `verifier.md`, `review.md`.                                                                                                                                                                                                                                                                                                            |
| **T14: Anti-drift anchors**             | AC-16               | For each active agent, verify Anti-Drift Anchor references isolated memory (not "do NOT write to `memory.md`" without qualification). No aggregator references.                                                                                                                                                                                                                                                                                                |
| **T15: Feature workflow**               | AC-17               | Verify `feature-workflow.prompt.md` has no aggregator, `analysis.md`, `design_critical_review.md`, `verifier.md`, or `review.md` references. Describes isolated memory model.                                                                                                                                                                                                                                                                                  |
| **T16: Convention adherence**           | AC-19               | For each modified agent, verify section ordering: YAML frontmatter → Title → Role → Prohibitions → Inputs → Outputs → Operating Rules → Workflow → Completion Contract → Anti-Drift Anchor. Verify Operating Rules 1–4 text is identical.                                                                                                                                                                                                                      |
| **T17: Completion contracts preserved** | AC-20               | For each sub-agent, verify completion contract format unchanged (DONE/ERROR or DONE/NEEDS_REVISION/ERROR). No new fields in completion line.                                                                                                                                                                                                                                                                                                                   |

### Integration-Level Verification

| Test                              | Scenario                    | Verification                                                                                                                                                                                                                                                                                                    |
| --------------------------------- | --------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **T18: Happy path flow**          | Full pipeline, no revisions | Trace Flow 1 from feature.md: Steps 0→1.1→2→3→3b→4→5→6→7. Verify every agent produces isolated memory (with artifact index including section pointers), orchestrator merges at each boundary, downstream agents read upstream memories before artifacts (memory-first reading), no combined artifacts produced. |
| **T19: CT revision flow**         | CT finds Critical severity  | Trace Flow 2: CT memories show Critical → orchestrator routes to designer → designer reads CT memories → consults artifact indexes → targeted reads of `ct-review/ct-*.md` sections → revision → CT re-run.                                                                                                     |
| **T20: V replan flow**            | V cluster NEEDS_REVISION    | Trace Flow 3: V memories show NEEDS_REVISION → planner reads V memories → consults artifact indexes → targeted reads of `verification/v-*.md` failure sections → cross-references failures → fix → V cluster re-run.                                                                                            |
| **T21: R-Security override**      | R-Security finds Blocker    | Orchestrator reads r-security memory → Highest Severity == Blocker → pipeline ERROR.                                                                                                                                                                                                                            |
| **T22: R-Knowledge non-blocking** | R-Knowledge returns ERROR   | Orchestrator ignores r-knowledge ERROR → pipeline result determined by remaining R sub-agents.                                                                                                                                                                                                                  |

---

## Tradeoffs & Alternatives Considered

### Tradeoff 1: Loss of Deduplication

**Decision:** Accept loss of cross-agent deduplication.
**Rationale:** Aggregators previously deduplicated findings across sub-agents (e.g., two CT agents flagging the same issue). Without aggregators, duplicate findings remain in individual artifact files. This is acceptable because: (a) the orchestrator only reads compact memories for routing — duplicates don't affect severity-based routing decisions; (b) downstream consumers reading full artifacts can identify duplicates themselves; (c) deduplication logic would add significant complexity to the orchestrator, violating NFR-1. Per Assumption A-4.

### Tradeoff 2: Loss of Unresolved Tensions Detection

**Decision:** Accept loss of contradiction detection.
**Rationale:** Aggregators cross-checked sub-agents for contradictions and surfaced them as "Unresolved Tensions." This detection is lost. Acceptable because: (a) tensions are a quality enhancement, not a safety feature; (b) downstream agents reading full artifacts can identify contradictions; (c) the orchestrator's routing decisions are severity-based, not contradiction-based. Per Assumption A-5.

### Tradeoff 3: Planner Self-Assembles Failure Mapping vs. Orchestrator Compiles It

**Decision:** Planner reads V memories first, then targeted V artifact sections (option b), **with explicit cross-referencing workflow instructions added to the planner prompt** (not deferred).
**Rationale:** The planner in replan mode reads V sub-agent memories for orientation and artifact index, then performs targeted reads of `verification/v-tasks.md` for failing task IDs, `verification/v-tests.md` for test failures, and `verification/v-feature.md` for unmet criteria, self-assembling the failure-to-task mapping. This aligns with the memory-first reading philosophy and avoids adding cross-referencing logic to the orchestrator.
**Mitigation (concrete, not deferred):** The planner's Tier 3 changes include a new "Replan Cross-Referencing Steps" workflow subsection (see APIs & Interfaces → Downstream Input Changes → planner.agent.md) that provides explicit step-by-step instructions for correlating task failures with test failures and unmet criteria. This replaces the v-aggregator's Step 5 ("Map Failures to Task IDs") with equivalent guidance embedded in the planner prompt. The cross-referencing steps produce a prioritized task list identical in function to the v-aggregator's Actionable Items table. Per Assumption A-6.

### Tradeoff 4: Uniform Branch-Merge Model for Sequential Agents

**Decision:** Sequential agents (spec, designer, planner) also use isolated memory files.
**Rationale:** Sequential agents could safely write directly to `memory.md` (no concurrency risk). Using isolated memory for them adds one file-per-agent but maintains a uniform model where the orchestrator is the sole `memory.md` writer. This simplifies the memory write safety rule and avoids special-casing. Per Assumption A-2.

### Tradeoff 5: Merge at Cluster Boundaries vs. Per-Agent

**Decision:** Merge all isolated memories after each cluster completes, not after each individual agent.
**Rationale:** Merging per-agent would provide incrementally fresher memory but adds complexity (merge after every single agent return). Merging at cluster boundaries is simpler and sufficient — within a parallel cluster, agents don't benefit from each other's memory anyway (they run concurrently). For sequential agents, the orchestrator merges immediately after each returns (effectively per-agent, since clusters of 1). Per Assumption A-3.

### Alternative Considered: Keep Aggregators, Add Memory Files

**Rejected.** Keeping aggregators while adding isolated memory files would add complexity without removing the bottleneck. The aggregator step would remain sequential, and the new memory files would be redundant with the existing combined artifacts.

### Alternative Considered: Extract Decision Logic to Separate Reference File

**Conditionally planned.** If the post-implementation orchestrator exceeds 450 lines (the firm threshold defined in NFR-1), the CT/V/R decision tables MUST be extracted to `dispatch-patterns.md` (or a new `cluster-decisions.md`) as a blocking follow-up. Given the revised estimate of +30–60 lines net growth (pushing the orchestrator to ~430–460 lines), extraction is likely needed and should be treated as a planned conditional step, not a deferred contingency. The extraction would move ~35 lines of decision tables to an external reference, with the orchestrator containing a short pointer: "Apply CT/V/R decision logic per `dispatch-patterns.md` section _Cluster Decision Tables_." This reduces the orchestrator's cognitive density while keeping decision logic version-controlled.

---

## Implementation Checklist & Deliverables

### Tier 1 — Files to Deprecate (3 files)

| File                     | Change                          | Acceptance Criteria |
| ------------------------ | ------------------------------- | ------------------- |
| `ct-aggregator.agent.md` | Replace with deprecation notice | AC-1                |
| `v-aggregator.agent.md`  | Replace with deprecation notice | AC-1                |
| `r-aggregator.agent.md`  | Replace with deprecation notice | AC-1                |

### Tier 2 — Major Changes (2 files)

| File                    | Changes                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Acceptance Criteria                                            |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| `orchestrator.agent.md` | (1) Update Outputs: add `memory/` directory, clarify sole writer to `memory.md`. (2) Update Global Rule 6: add Merge action, update prune checkpoints. (3) Update Global Rule 10: approval gate reference. (4) Update Global Rule 12: orchestrator sole writer, remove aggregator lists. (5) Update Documentation Structure: add `memory/`, remove `analysis.md`, `design_critical_review.md`, `verifier.md`, `review.md`. (6) Update cluster dispatch pattern references: remove aggregator steps. (7) Remove Step 1.2 (synthesis). (8) Update Step 2 inputs: researcher memories as primary input, `research/*.md` sections for selective reads (instead of `analysis.md`). (9) Update Step 3 inputs: spec + researcher memories as primary input, selective artifact reads. (10) Replace Step 3b.2 (CT aggregator invocation) with orchestrator CT decision logic (include self-verification logging). (11) Update Step 3b.3: reference CT memories, not `design_critical_review.md`; **replace "forward Unresolved Tensions as planning constraints" with "forward all High/Critical findings from individual CT artifacts as planning constraints"** (Unresolved Tensions concept is removed with aggregators). (12) Update Step 4 inputs: designer + spec memories as primary input, selective reads of `design.md`, `feature.md`, `research/*.md` sections; remove `design_critical_review.md`/`analysis.md`; add CT memories → selective `ct-review/ct-*.md` reads. (13) Update Step 5.2: orchestrator merges isolated memories between waves. (14) Replace Step 6.3 (V aggregator invocation) with orchestrator V decision logic (include self-verification logging). (15) Update Step 6.4: Pattern C uses V memories; **planner invocation includes explicit `MODE: REPLAN` signal** with paths to V memory files (`memory/v-*.mem.md`). (16) Replace Step 7.3 (R aggregator invocation) with orchestrator R decision logic (include self-verification logging and severity mapping note). (17) Update Step 7.4: R memories, not `review.md`. (18) Update NEEDS_REVISION Routing Table: replace aggregator names with orchestrator evaluation; **remove/replace "forward Unresolved Tensions as planning constraints" row text**. (19) Update Orchestrator Expectations table: remove aggregator rows, update sub-agent rows. (20) Update Memory Lifecycle Actions: add Merge, update Validate. (21) Update Parallel Execution Summary: remove aggregator/synthesis steps, add merge steps. (22) Update Completion Contract: "R cluster returns DONE" (not "R Aggregator"). (23) Update Anti-Drift Anchor: add memory merge, remove aggregator pattern reference. (24) Add `memory/` directory creation to Step 0. (25) Add isolated memory file as orchestrator output (`memory/orchestrator.mem.md` is NOT produced — orchestrator does not write its own memory). (26) **Add self-verification logging instructions** to each cluster decision flow: orchestrator logs severity values found and routing decision in `memory.md` Recent Updates before proceeding. | AC-2, AC-3, AC-6, AC-7, AC-8, AC-9, AC-13, AC-14, AC-15, AC-18 |
| `researcher.agent.md`   | (1) Remove synthesis mode (Mode 2) entirely: inputs, outputs, workflow, completion contract. (2) Remove `analysis.md` from outputs. (3) Update description: focused mode only. (4) Add `memory/researcher-<focus>.mem.md` to outputs. (5) Add "Write Isolated Memory" workflow step. (6) Update Anti-Drift Anchor.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | AC-4, AC-5                                                     |

### Tier 3 — Moderate Changes (6 files)

| File                         | Changes                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Acceptance Criteria      |
| ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------ |
| `spec.agent.md`              | (1) Replace `analysis.md` input with researcher memory files (`memory/researcher-*.mem.md`) as primary inputs + `research/*.md` for selective section reads. (2) Add `memory/spec.mem.md` to outputs. (3) Add "Write Isolated Memory" workflow step (with artifact index including section pointers). (4) Replace "Update `memory.md`" step with "Write Isolated Memory". (5) Add memory-first reading workflow: read researcher memories → consult artifact indexes → targeted reads of research artifact sections. (6) Update Anti-Drift Anchor.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | AC-3, AC-5, AC-10        |
| `designer.agent.md`          | (1) Replace `analysis.md` input with spec + researcher memory files as primary inputs + `research/*.md` for selective section reads. (2) Replace `design_critical_review.md` input with CT memory files (`memory/ct-*.mem.md`) as primary inputs + `ct-review/ct-*.md` for selective section reads. (3) Add `memory/designer.mem.md` to outputs. (4) Add "Write Isolated Memory" workflow step (with artifact index including section pointers). (5) Replace "Update `memory.md`" step with "Write Isolated Memory". (6) Add memory-first reading workflow: read spec + researcher memories → consult artifact indexes → targeted reads of feature.md + research artifact sections; in revision mode: read CT memories → targeted reads of ct-review sections. (7) Update Anti-Drift Anchor.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | AC-3, AC-5, AC-10, AC-11 |
| `planner.agent.md`           | (1) Replace `analysis.md` input with designer + spec memory files as primary inputs + `design.md`, `feature.md`, `research/*.md` for selective section reads. (2) Replace `verifier.md` (replan) with V memory files (`memory/v-*.mem.md`) as primary inputs + `verification/v-tasks.md`, `verification/v-tests.md`, `verification/v-feature.md` for selective section reads. (3) Remove `design_critical_review.md` input, add CT memory files → selective `ct-review/ct-*.md` reads (if present). (4) Add `memory/planner.mem.md` to outputs. (5) Add "Write Isolated Memory" workflow step (with artifact index including section pointers). (6) Replace "Update `memory.md`" step with "Write Isolated Memory". (7) **Replace replan mode detection:** change from "`verifier.md` exists with failures" to "orchestrator provides `MODE: REPLAN` signal with V memory paths" (primary) + "fallback: `verification/v-tasks.md` exists with failing task IDs" (secondary). (8) **Add Replan Cross-Referencing Steps** workflow subsection: memory-first preamble (read V memories, consult artifact indexes) + 5-step procedure for targeted reads of v-tasks.md, v-tests.md, v-feature.md sections and producing a prioritized remediation task list. (9) Add memory-first reading workflow for normal mode: read designer + spec memories → consult artifact indexes → targeted reads of design/feature/research sections. (10) Update Anti-Drift Anchor.                                                          | AC-3, AC-5, AC-10, AC-12 |
| `r-security.agent.md`        | (1) **Promote from Tier 4 — targeted changes beyond generic sub-agent updates.** (2) Add `memory/r-security.mem.md` to outputs. (3) Add "Write Isolated Memory" workflow step. (4) **Rewrite Pipeline Blocker Override Rule section** (current lines 61–67): replace "the R Aggregator MUST override its aggregated result to `ERROR`" with "the orchestrator reads the `Highest Severity` field in `memory/r-security.mem.md` and declares pipeline `ERROR` if `Blocker` is found." Emphasize that the `Highest Severity` memory field is now the sole vehicle for communicating blocker status to the orchestrator. (5) **Update completion contract guidance** (current line 224): replace "the R Aggregator will override the pipeline result to ERROR" with "the orchestrator will read your `Highest Severity` memory field and declare pipeline ERROR if `Blocker` is found. You MUST populate `Highest Severity` with `Blocker` (not `Critical`) when any finding has pipeline-blocking severity." (6) **Update Anti-Drift Anchor** (current line 229): keep "Your Critical/Blocker findings block the entire pipeline" but add: "via the `Highest Severity` field in your isolated memory file." Replace any aggregator references. (7) **Constrain severity vocabulary:** Add explicit instruction: "In `Highest Severity`, use the R cluster canonical taxonomy: `Blocker/Major/Minor`. Do not use `Critical` — use `Blocker` instead." (8) Remove "No Memory Write" step, replace with the new write step. | AC-5, AC-9, AC-16        |
| `dispatch-patterns.md`       | (1) Pattern A: remove steps 4 (invoke aggregator), 6 (check aggregator contract). Replace with "Orchestrator reads sub-agent isolated memories and determines cluster result". Keep ≥2 outputs threshold. (2) Pattern B: remove steps 3 (forward to aggregator), 6 (invoke aggregator), 7 (check aggregator contract). Replace with orchestrator reads memories.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | AC-13                    |
| `feature-workflow.prompt.md` | (1) Remove aggregator references from Rules. (2) Update research description: no synthesis step. (3) Add isolated memory model description. (4) Update Key Artifacts table: remove `analysis.md`, `design_critical_review.md`, `verifier.md`, `review.md` references (if any). (5) Update approval gate reference: "after research completion" (not "research synthesis").                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | AC-17                    |

### Tier 4 — Minor Changes (13 files — add isolated memory output)

All 13 files receive the same set of changes:

1. **Add to Outputs:** `memory/<agent-name>.mem.md`
2. **Add workflow step:** "Write Isolated Memory" (template from Data Models section)
3. **Replace "No Memory Write" step** (if exists) with the new write step
4. **Update Anti-Drift Anchor:** Replace "do NOT write to `memory.md`" / aggregator references with "write only to isolated memory file"
5. **Remove aggregator references** (if any — e.g., "The V-Aggregator handles memory updates")

| File                          | Memory File Name                   | Additional Changes                                                             |
| ----------------------------- | ---------------------------------- | ------------------------------------------------------------------------------ |
| `ct-security.agent.md`        | `memory/ct-security.mem.md`        | —                                                                              |
| `ct-scalability.agent.md`     | `memory/ct-scalability.mem.md`     | —                                                                              |
| `ct-maintainability.agent.md` | `memory/ct-maintainability.mem.md` | —                                                                              |
| `ct-strategy.agent.md`        | `memory/ct-strategy.mem.md`        | —                                                                              |
| `v-build.agent.md`            | `memory/v-build.mem.md`            | —                                                                              |
| `v-tests.agent.md`            | `memory/v-tests.mem.md`            | —                                                                              |
| `v-tasks.agent.md`            | `memory/v-tasks.mem.md`            | Remove "for the V-Aggregator and planner" → "for the orchestrator and planner" |
| `v-feature.agent.md`          | `memory/v-feature.mem.md`          | Remove "for the V-Aggregator and planner" → "for the orchestrator and planner" |
| `r-quality.agent.md`          | `memory/r-quality.mem.md`          | —                                                                              |

| `r-testing.agent.md` | `memory/r-testing.mem.md` | — |
| `r-knowledge.agent.md` | `memory/r-knowledge.mem.md` | R-Knowledge `verifier.md` reference in memory navigation → remove or replace with individual V artifacts |
| `implementer.agent.md` | `memory/implementer-<task-id>.mem.md` | Task ID included in memory file name; add memory-first reading workflow: read planner memory → consult artifact index → targeted reads of task file + design sections |
| `documentation-writer.agent.md` | `memory/documentation-writer-<task-id>.mem.md` | Task ID included in memory file name; replace "Do NOT write to `memory.md`" |

### Tier 5 — No Changes (1 file)

| File                        | Reason                                       |
| --------------------------- | -------------------------------------------- |
| `critical-thinker.agent.md` | Already deprecated (C-3). No changes needed. |

### Acceptance Criteria Coverage Matrix

| AC    | Verified By | Implementation Tier               |
| ----- | ----------- | --------------------------------- |
| AC-1  | T1          | Tier 1                            |
| AC-2  | T2          | Tiers 1–4                         |
| AC-3  | T3          | Tiers 2–3                         |
| AC-4  | T4          | Tier 2 (researcher)               |
| AC-5  | T5          | Tiers 2–4                         |
| AC-6  | T6          | Tier 2 (orchestrator)             |
| AC-7  | T7          | Tier 2 (orchestrator)             |
| AC-8  | T8          | Tier 2 (orchestrator)             |
| AC-9  | T9          | Tier 2 (orchestrator)             |
| AC-10 | T10         | Tier 3 (spec, designer, planner)  |
| AC-11 | T10         | Tier 3 (designer)                 |
| AC-12 | T10         | Tier 3 (planner)                  |
| AC-13 | T11         | Tier 3 (dispatch-patterns)        |
| AC-14 | T12         | Tier 2 (orchestrator)             |
| AC-15 | T13         | Tier 2 (orchestrator)             |
| AC-16 | T14         | Tiers 2–4                         |
| AC-17 | T15         | Tier 3 (feature-workflow)         |
| AC-18 | T4          | Tier 2 (orchestrator, researcher) |
| AC-19 | T16         | All tiers                         |
| AC-20 | T17         | Tier 4                            |

---

## Revision Notes (Post-CT Review)

This section documents revisions made to address **5 High-severity findings** from the Critical Thinking review (ct-security, ct-scalability, ct-maintainability, ct-strategy).

### Revision 1: R-Security Pipeline Blocker Override — Tier Promotion (CT-Security Finding 1)

**Problem:** R-Security has unique pipeline-blocker language referencing the "R Aggregator" as the enforcement mechanism. The original design treated r-security as a generic Tier 4 change, but its `Pipeline Blocker Override Rule` section and completion contract require targeted updates that go beyond adding an isolated memory output.

**Changes made:**

- Moved `r-security.agent.md` from Tier 4 to **Tier 3** with 8 explicit change instructions.
- Added targeted rewrite instructions for the Pipeline Blocker Override Rule section (lines 61–67), completion contract guidance (line 224), and Anti-Drift Anchor (line 229).
- Added explicit severity vocabulary constraint: R-Security MUST use "Blocker" (not "Critical") in the `Highest Severity` memory field.
- Added emphasis that the `Highest Severity` field is the sole vehicle for communicating blocker status to the orchestrator.
- Updated Tier 4 count from 14 to 13 files.
- Updated Tier 3 count from 5 to 6 files.

### Revision 2: Planner Replan Mode Detection (CT-Security Finding 2)

**Problem:** The planner currently detects replan mode via `verifier.md` existence. Removing `verifier.md` breaks this detection, and the original design did not define a replacement mechanism.

**Changes made:**

- Added a new "Planner Replan Mode Detection" subsection to APIs & Interfaces → Downstream Input Changes → planner.agent.md.
- Defined a two-part replacement mechanism:
  1. **Primary:** Orchestrator dispatches planner with explicit `MODE: REPLAN` signal plus V artifact paths.
  2. **Secondary (redundant safety):** Planner checks if `verification/v-tasks.md` exists with failing task IDs.
- Updated pipeline flow Step 6.4 to include the explicit `MODE: REPLAN` signal in planner invocation.
- Updated orchestrator Tier 2 change item 15 to include the mode signal.
- Updated planner Tier 3 changes to include the new mode detection logic (item 7).

### Revision 3: Task-Failure Cross-Referencing (CT-Security Finding 3 + CT-Scalability Finding 2)

**Problem:** The v-aggregator's structured Actionable Items and task-failure mapping were lost. The planner must self-assemble this from 3 V artifacts, but the original design deferred the mitigation ("monitor in early usage").

**Changes made:**

- Added a new "Planner Replan Cross-Referencing Workflow" section to APIs & Interfaces → Downstream Input Changes → planner.agent.md.
- Defined 5 explicit workflow steps: extract failing task IDs from v-tasks.md, correlate test failures from v-tests.md, map unmet criteria from v-feature.md, prioritize tasks, produce targeted remediation plan.
- Updated Tradeoff 3 to indicate the mitigation is concrete (not deferred): the planner receives explicit cross-referencing instructions.
- Updated the Failure & Recovery table entry for "Planner replan without aggregated task-failure mapping" to reference the new cross-referencing workflow.
- Updated planner Tier 3 changes to include the cross-referencing steps (item 8).

### Revision 4: Orchestrator Complexity Transfer (CT-Scalability Finding 1)

**Problem:** The "+10 lines net" estimate for orchestrator growth was misleadingly low. It counted lines but ignored the cognitive complexity density of absorbing ~750 lines of aggregator logic into compact decision tables.

**Changes made:**

- Revised NFR-1 with honest estimate: **+30–60 lines net** (pushing orchestrator to ~430–460 lines), with explicit breakdown.
- Added "Complexity density note" acknowledging that ~35 lines of compact decision tables carry the cognitive weight of ~750 lines of aggregator procedures.
- Added **self-verification logging steps** to all three decision flows (CT, V, R): the orchestrator logs severity values and routing decisions in memory.md before proceeding, providing auditability.
- Changed the extraction threshold from a vague contingency ("if the orchestrator exceeds ~450 lines") to a **firm 450-line threshold** with a blocking follow-up task.
- Updated the "Alternative Considered: Extract Decision Logic" section to reflect the conditional plan rather than deferral.
- Updated orchestrator Tier 2 changes to include self-verification logging (new item 26).

### Revision 5: Severity Taxonomy Confusion (CT-Security Finding 4)

**Problem:** R-Security uses both "Blocker" and "Critical" terminology (line 66: `Severity: Blocker (Critical)`). The memory template listed `Blocker/Major/Minor` but the R decision flow checked for "Blocker or Critical," creating ambiguity in the most safety-critical routing decision.

**Changes made:**

- Updated memory template R agent comment to explicitly state R-Security MUST use "Blocker" (not "Critical").
- Updated R-Security cluster-specific field description to constrain vocabulary to `Blocker/Major/Minor` only, noting that "Critical" is not a valid R taxonomy value.
- Updated R Cluster Decision Flow step 3: now checks only for "Blocker" (not "Blocker or Critical"), with a note that "Critical" is treated as "Blocker" for safety but indicates a prompt compliance gap.
- Updated NFR-5 to document the taxonomy distinction and the rationale for constraining R-Security's vocabulary.
- Added severity vocabulary constraint to r-security's Tier 3 change instructions (item 7).

### Additional Changes

- **Unresolved Tensions escalation path:** Updated pipeline flow Step 3b.r and orchestrator Tier 2 change items 11 and 18 to replace "forward Unresolved Tensions as planning constraints" (which references a concept that no longer exists post-aggregator removal) with "forward all High/Critical findings from individual CT artifacts as planning constraints."
- **Orchestrator Tier 2 changes:** Expanded from 25 to 26 items to include self-verification logging (item 26) and explicit handling of the Unresolved Tensions replacement and MODE: REPLAN signal.

### Revision 6: Memory-First Reading Pattern for Downstream Agents

**Problem:** The original design had downstream agents reading full upstream artifact files directly (e.g., spec reads all 4 `research/*.md` files, designer reads all `ct-review/ct-*.md` files). This contradicts the efficiency intent of the memory system — memories exist as compact summaries with artifact pointers, but downstream agents bypassed them entirely when consuming upstream outputs.

**Design Principle:** Memories serve as an index/summary — analogous to a git log vs. reading full diffs. Agents read the log (memories) first, then `git show` (targeted read) only the specific sections they need. No agent should read a full artifact file unless the memory index tells it to.

**Changes made:**

1. **Architecture diagram ("After"):** Updated downstream agent flow from "reads primary artifacts directly (by path)" to "reads upstream memories → consults artifact index → targeted artifact reads."

2. **Boundary Decisions:** Added third bullet — "Memory-first reading for all downstream agents" — establishing the pattern as a core architectural boundary. Updated second bullet to remove "directly."

3. **Memory file template:** Renamed `Artifacts Produced` section to **`Artifact Index`** with section-level pointers using `§Section` notation and brief relevance notes. This transforms the memory from a simple file list into a navigable index that downstream agents use for targeted reads.

4. **Orchestrator merge protocol:** Updated merge extraction logic to reference `Artifact Index` (not `Artifacts Produced`) and preserve section-level granularity when merging into shared `memory.md`. The shared `memory.md` Artifact Index (which already had a "Key Sections" column) becomes the primary navigation tool for all downstream agents.

5. **Write Isolated Memory workflow step:** Updated the universal agent workflow step to produce `Artifact Index` with section-level pointers instead of a flat file list.

6. **Downstream Input Changes (API & Interfaces):** Complete rewrite of spec, designer, and planner input specifications:
   - Each agent now lists **memory files as primary inputs** (read first) and **artifact files as selective inputs** (read only sections referenced by memory artifact indexes).
   - Added explicit **Reading workflow** descriptions for each agent showing the memory → artifact index → targeted read flow.
   - Added **implementer input specification** (new): reads planner memory → consults artifact index → targeted reads of task file + design sections.

7. **Planner Replan Mode Detection:** Updated orchestrator dispatch to provide V **memory** paths (not artifact paths). Updated Step 6.4 reference.

8. **Planner Replan Cross-Referencing Workflow:** Added memory-first preamble. Added Step 0: read V sub-agent memories and artifact indexes before performing targeted reads. Updated Steps 1–3 to reference "targeted sections" (per artifact index) instead of reading full files.

9. **Pipeline Flow:** Updated Steps 2, 3, 3b.r, 4, 5, and 6.4 to show memory-first reading at each stage (e.g., "Spec reads researcher memories → consults artifact indexes → targeted reads" instead of "Spec ← research/\*.md").

10. **Orchestrator Memory Merge Sequence:** Updated step 3a to extract `Artifact Index entries (with section pointers)` instead of `Artifacts Produced`.

11. **Removed Artifacts table:** Updated downstream impact column to describe memory-first navigation (e.g., "read researcher memories → targeted reads of sections") instead of "read directly."

12. **Implementation Checklist:**
    - **Tier 2 orchestrator:** Updated items 8, 9, 12, 15 to reference memory-first inputs and V memory paths.
    - **Tier 3 spec/designer/planner:** Rewritten to specify memory files as primary inputs, artifact files for selective reads, artifact index with section pointers, and memory-first reading workflows.
    - **Tier 4 implementer:** Added memory-first reading workflow note.

13. **Testing Strategy:** Updated T10 (verify memory-first input pattern, not direct artifact reads), T18 (verify artifact indexes and memory-first reading in happy path), T19 (designer reads CT memories before artifacts), T20 (planner reads V memories before artifacts).

14. **Tradeoff 3:** Updated decision and rationale to describe memory-first reading rather than "reads individual V artifacts directly."

**Constraints preserved:**

- Memory files remain compact (≤30 lines, NFR-2). The `Artifact Index` with section pointers replaces the flat `Artifacts Produced` list within the same line budget.
- Artifact files still exist and are still created by each agent (unchanged).
- Agents retain access to full artifacts if needed — but they navigate via memory first, reading only the sections the artifact index points to.
- The memory IS the index — it tells you what's in the artifact and where.
