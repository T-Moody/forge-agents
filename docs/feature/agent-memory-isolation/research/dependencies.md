# Research: Dependencies

## Focus Area

dependencies

## Summary

The pipeline has a hub-and-spoke data flow where aggregator agents (ct-aggregator, v-aggregator, r-aggregator, researcher synthesis mode) serve as sequential merge points between parallel sub-agent clusters and downstream consumers; removing them requires the orchestrator to absorb their completion-contract logic, memory-write responsibilities, and combined-artifact production, while downstream agents must shift from reading single combined files (analysis.md, design_critical_review.md, verifier.md, review.md) to reading individual sub-agent artifact files directly.

## Findings

### Current Data Flow

The pipeline is an 8-step sequential flow with parallelism within steps. Data flows through explicit file artifacts on disk; agents read input files and write output files. The orchestrator coordinates but never touches artifact content — it only manages `memory.md` lifecycle and dispatches agents.

**Pipeline data flow (current):**

```
Step 0: Orchestrator → initial-request.md, memory.md
Step 1.1: researcher ×4 (parallel) → research/architecture.md, research/impact.md, research/dependencies.md, research/patterns.md
Step 1.2: researcher (synthesis) ← research/*.md → analysis.md + memory.md write
Step 2: spec ← analysis.md → feature.md + memory.md write
Step 3: designer ← analysis.md, feature.md → design.md + memory.md write
Step 3b.1: CT ×4 (parallel) ← design.md, feature.md → ct-review/ct-*.md
Step 3b.2: ct-aggregator ← ct-review/ct-*.md, design.md → design_critical_review.md + memory.md write
Step 3b.3: [conditional] designer revision ← design_critical_review.md → design.md update
Step 4: planner ← analysis.md, feature.md, design.md, [design_critical_review.md] → plan.md, tasks/*.md + memory.md write
Step 5: implementer/doc-writer ×N (parallel waves) ← task file, feature.md, design.md → code/tests/docs
Step 5 between-waves: orchestrator extracts Lessons Learned → memory.md write
Step 6.1: v-build ← codebase → verification/v-build.md
Step 6.2: v-tests, v-tasks, v-feature (parallel) ← v-build.md + others → verification/v-*.md
Step 6.3: v-aggregator ← verification/v-*.md → verifier.md + memory.md write
Step 6.4: [conditional] Pattern C replan loop using verifier.md
Step 7.2: R ×4 (parallel) ← initial-request.md, git diff, etc. → review/r-*.md + knowledge-suggestions.md + decisions.md
Step 7.3: r-aggregator ← review/r-*.md, knowledge-suggestions.md → review.md + memory.md write
```

**Key data flow properties:**

- All agents read `memory.md` first for orientation (memory-first protocol).
- Parallel sub-agents are read-only with respect to `memory.md`.
- Only aggregators and sequential agents (spec, designer, planner) write to `memory.md`.
- Aggregators produce combined artifact files that downstream agents consume.
- The orchestrator reads completion contracts (DONE/NEEDS_REVISION/ERROR) from aggregators to make routing decisions.

### Agent Input/Output Map

| Agent                       | Reads (Inputs)                                                                                                | Writes (Outputs)                                              | Writes to memory.md                | Completion Contract           |
| --------------------------- | ------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------- | ---------------------------------- | ----------------------------- |
| **orchestrator**            | User request, all completion contracts                                                                        | initial-request.md, memory.md (lifecycle)                     | Yes (init, prune, extract lessons) | N/A (top-level)               |
| **researcher (focused ×4)** | memory.md, initial-request.md, codebase                                                                       | research/{architecture,impact,dependencies,patterns}.md       | No (read-only)                     | DONE / ERROR                  |
| **researcher (synthesis)**  | memory.md, initial-request.md, research/\*.md                                                                 | analysis.md                                                   | Yes (append)                       | DONE / ERROR                  |
| **spec**                    | memory.md, initial-request.md, analysis.md                                                                    | feature.md                                                    | Yes (append)                       | DONE / ERROR                  |
| **designer**                | memory.md, initial-request.md, analysis.md, feature.md, [design_critical_review.md]                           | design.md                                                     | Yes (append)                       | DONE / ERROR                  |
| **ct-security**             | memory.md, initial-request.md, design.md, feature.md                                                          | ct-review/ct-security.md                                      | No (read-only)                     | DONE / ERROR                  |
| **ct-scalability**          | memory.md, initial-request.md, design.md, feature.md                                                          | ct-review/ct-scalability.md                                   | No (read-only)                     | DONE / ERROR                  |
| **ct-maintainability**      | memory.md, initial-request.md, design.md, feature.md                                                          | ct-review/ct-maintainability.md                               | No (read-only)                     | DONE / ERROR                  |
| **ct-strategy**             | memory.md, initial-request.md, design.md, feature.md                                                          | ct-review/ct-strategy.md                                      | No (read-only)                     | DONE / ERROR                  |
| **ct-aggregator**           | memory.md, ct-review/ct-\*.md, design.md                                                                      | design_critical_review.md                                     | Yes (append)                       | DONE / NEEDS_REVISION / ERROR |
| **planner**                 | memory.md, initial-request.md, analysis.md, feature.md, design.md, [verifier.md], [design_critical_review.md] | plan.md, tasks/\*.md                                          | Yes (append)                       | DONE / ERROR                  |
| **implementer**             | memory.md, task file, feature.md, design.md                                                                   | Code files, test files, task file (status update)             | No (read-only)                     | DONE / ERROR                  |
| **documentation-writer**    | memory.md, task file, feature.md, design.md, codebase                                                         | Documentation files, task file (status update)                | No (read-only)                     | DONE / ERROR                  |
| **v-build**                 | memory.md, codebase                                                                                           | verification/v-build.md                                       | No (read-only)                     | DONE / ERROR                  |
| **v-tests**                 | memory.md, v-build.md, codebase                                                                               | verification/v-tests.md                                       | No (read-only)                     | DONE / NEEDS_REVISION / ERROR |
| **v-tasks**                 | memory.md, v-build.md, plan.md, tasks/\*.md, codebase                                                         | verification/v-tasks.md                                       | No (read-only)                     | DONE / NEEDS_REVISION / ERROR |
| **v-feature**               | memory.md, v-build.md, feature.md, design.md, codebase                                                        | verification/v-feature.md                                     | No (read-only)                     | DONE / NEEDS_REVISION / ERROR |
| **v-aggregator**            | memory.md, verification/v-\*.md                                                                               | verifier.md                                                   | Yes (append)                       | DONE / NEEDS_REVISION / ERROR |
| **r-quality**               | memory.md, initial-request.md, design.md, git diff, codebase                                                  | review/r-quality.md                                           | No (read-only)                     | DONE / NEEDS_REVISION / ERROR |
| **r-security**              | memory.md, initial-request.md, git diff, codebase                                                             | review/r-security.md                                          | No (read-only)                     | DONE / NEEDS_REVISION / ERROR |
| **r-testing**               | memory.md, initial-request.md, feature.md, git diff, codebase                                                 | review/r-testing.md                                           | No (read-only)                     | DONE / NEEDS_REVISION / ERROR |
| **r-knowledge**             | memory.md, initial-request.md, decisions.md, .github/instructions/, git diff, codebase                        | review/r-knowledge.md, knowledge-suggestions.md, decisions.md | No (read-only)                     | DONE / ERROR                  |
| **r-aggregator**            | memory.md, review/r-\*.md, knowledge-suggestions.md                                                           | review.md                                                     | Yes (append)                       | DONE / NEEDS_REVISION / ERROR |

### Aggregator Responsibilities Being Removed

Four aggregator/synthesis roles are being removed. Each has specific responsibilities that must be redistributed:

#### 1. Researcher Synthesis Mode (Step 1.2)

**Current responsibilities:**

- Reads ALL partial research files from `research/` directory
- Merges, deduplicates, and organizes findings into coherent `analysis.md`
- Resolves contradictions between partial analyses
- Preserves file paths and line numbers from partials
- Writes to `memory.md` (Artifact Index + Recent Updates)
- Returns DONE / ERROR

**Downstream consumers of `analysis.md`:**

- `spec` (reads analysis.md as primary input — line: "Read `analysis.md` thoroughly")
- `designer` (reads analysis.md — line: "Read `analysis.md` and `feature.md` thoroughly")
- `planner` (reads analysis.md — listed as explicit input)

**Impact of removal:** Spec, designer, and planner currently read ONE combined file. Without synthesis, they must read 4 individual research files (research/architecture.md, research/impact.md, research/dependencies.md, research/patterns.md). This increases their input file count from 1 to 4 but eliminates the synthesis bottleneck.

#### 2. CT-Aggregator (Step 3b.2)

**Current responsibilities:**

- Reads all 4 CT sub-agent output files
- Validates inputs (≥2 sub-agent outputs required)
- Collects, deduplicates, and sorts findings by severity
- Synthesizes cross-cutting concerns
- Surfaces unresolved tensions (contradictions between sub-agents)
- Merges requirement coverage tables
- **Determines completion contract**: Critical/High → NEEDS_REVISION; Medium/Low only → DONE; <2 outputs → ERROR
- Writes `design_critical_review.md`
- Writes to `memory.md`

**Downstream consumers of `design_critical_review.md`:**

- `designer` (reads during revision cycle when CT returns NEEDS_REVISION)
- `planner` (reads planning constraints from it, if present)

**Impact of removal:** The orchestrator must absorb the completion contract logic (severity-based DONE/NEEDS_REVISION/ERROR decision). The `design_critical_review.md` combined artifact is no longer produced. Designer receives individual CT files directly. Planner reads individual CT files for planning constraints. Unresolved tension detection may be lost unless the orchestrator or designer handles it.

#### 3. V-Aggregator (Step 6.3)

**Current responsibilities:**

- Reads all 4 V sub-agent output files
- Validates inputs (2+ sub-agent outputs required for aggregation)
- Compiles unified verification report
- **Critical: Maps failures to task IDs** — the planner depends on this for targeted replanning
- Deduplicates issues across sub-agents
- Surfaces unresolved tensions
- **Determines completion contract** using a detailed decision table (V-Build status × V-Tests × V-Tasks × V-Feature → DONE/NEEDS_REVISION/ERROR)
- Writes `verifier.md` (with Actionable Items section)
- Writes to `memory.md`

**Downstream consumers of `verifier.md`:**

- Orchestrator (reads verdict for Pattern C replan loop)
- `planner` in replan mode (reads Actionable Items and Failing Task IDs for targeted replanning)

**Impact of removal:** This is the highest-risk removal. The planner's replan mode explicitly depends on `verifier.md`'s Actionable Items section with task ID failure mapping. Without the aggregator, the orchestrator must either: (a) produce the task ID failure mapping itself by reading individual V sub-agent outputs, or (b) the planner must read 4 individual V files and self-assemble the failure mapping. The complex completion contract decision table must move to the orchestrator.

#### 4. R-Aggregator (Step 7.3)

**Current responsibilities:**

- Reads all 4 R sub-agent output files + knowledge-suggestions.md
- **Enforces R-Security pipeline override** (R-Security ERROR/Critical → pipeline ERROR)
- Validates inputs (R-Security missing → immediate ERROR; 2+ non-knowledge missing → ERROR)
- Merges and deduplicates findings by severity
- Processes R-Knowledge (non-blocking)
- Extracts high-value knowledge suggestions
- Synthesizes cross-cutting concerns
- Surfaces unresolved tensions
- **Determines completion contract** with complex rules (security override → ERROR; 2+ missing → ERROR; Major finding → NEEDS_REVISION; else → DONE)
- Writes `review.md`
- Writes to `memory.md`

**Downstream consumers of `review.md`:**

- Orchestrator (reads verdict to determine workflow completion or NEEDS_REVISION routing)

**Impact of removal:** The R-Security pipeline override rule must move to the orchestrator. The orchestrator must read individual R sub-agent outputs and apply the security-first completion contract logic. The knowledge-suggestions.md pass-through and high-value extraction logic is lost unless handled by the orchestrator.

### Proposed Data Flow Changes

With aggregator removal and isolated memory, the data flow changes as follows:

**New data flow (proposed):**

```
Step 0: Orchestrator → initial-request.md, memory.md
Step 1.1: researcher ×4 (parallel) → research/*.md + isolated memory files
Step 1.2: Orchestrator reads researcher memories (NOT full artifacts) → merges to memory.md
         [NO synthesis step; NO analysis.md]
Step 2: spec ← research/*.md (4 files directly) → feature.md + isolated memory
Step 3: designer ← research/*.md, feature.md → design.md + isolated memory
Step 3b.1: CT ×4 (parallel) ← design.md, feature.md → ct-review/ct-*.md + isolated memories
Step 3b.2: Orchestrator reads CT memories → determines DONE/NEEDS_REVISION/ERROR
           [NO ct-aggregator; NO design_critical_review.md]
Step 3b.3: [conditional] designer revision ← individual ct-review/ct-*.md files
Step 4: planner ← research/*.md, feature.md, design.md, [ct-review/ct-*.md] → plan.md, tasks/*.md + isolated memory
Step 5: implementer/doc-writer ×N → code/tests/docs + isolated memories
Step 5 between-waves: Orchestrator reads implementer memories → merges to memory.md
Step 6.1: v-build → verification/v-build.md + isolated memory
Step 6.2: v-tests, v-tasks, v-feature (parallel) → verification/v-*.md + isolated memories
Step 6.3: Orchestrator reads V memories → determines DONE/NEEDS_REVISION/ERROR
          [NO v-aggregator; NO verifier.md]
Step 6.4: [conditional] Pattern C replan using individual V sub-agent files
Step 7.2: R ×4 (parallel) → review/r-*.md + isolated memories
Step 7.3: Orchestrator reads R memories → determines DONE/NEEDS_REVISION/ERROR
          [NO r-aggregator; NO review.md]
```

**Key changes to data paths:**

| Current Path                                                                       | Proposed Path                                                       | Files Affected                                             |
| ---------------------------------------------------------------------------------- | ------------------------------------------------------------------- | ---------------------------------------------------------- |
| research/\*.md → researcher synthesis → analysis.md → spec, designer, planner      | research/\*.md → spec, designer, planner (direct)                   | spec.agent.md, designer.agent.md, planner.agent.md         |
| ct-review/ct-\*.md → ct-aggregator → design_critical_review.md → designer, planner | ct-review/ct-\*.md → designer, planner (direct)                     | designer.agent.md, planner.agent.md, orchestrator.agent.md |
| verification/v-\*.md → v-aggregator → verifier.md → orchestrator, planner          | verification/v-\*.md → orchestrator, planner (direct)               | planner.agent.md, orchestrator.agent.md                    |
| review/r-\*.md → r-aggregator → review.md → orchestrator                           | review/r-\*.md → orchestrator (direct)                              | orchestrator.agent.md                                      |
| Sub-agent → shared memory.md (via aggregator write)                                | Sub-agent → isolated memory file → orchestrator merges to memory.md | All agents, orchestrator.agent.md                          |

### Completion Contract Migration

The following completion contract decisions currently made by aggregators must move to the orchestrator:

#### CT Cluster Completion (from ct-aggregator → orchestrator)

**Current logic (ct-aggregator.agent.md, Workflow step 10):**

- Any finding rated Critical or High severity → `NEEDS_REVISION`
- All findings Medium or Low → `DONE`
- <2 sub-agent outputs → `ERROR`

**Migration requirement:** Orchestrator must read CT sub-agent isolated memories (which should include severity summaries) to determine if any Critical/High findings exist. The orchestrator should NOT read full ct-review/ct-\*.md artifacts for this — only compact memory summaries per the "Orchestrator Reads Memories, Not Full Artifacts" rule.

#### V Cluster Completion (from v-aggregator → orchestrator)

**Current logic (v-aggregator.agent.md, Workflow step 7):**

| V-Build | V-Tests        | V-Tasks         | V-Feature       | Result                   |
| ------- | -------------- | --------------- | --------------- | ------------------------ |
| DONE    | DONE           | DONE            | DONE            | DONE                     |
| DONE    | NEEDS_REVISION | any             | any             | NEEDS_REVISION           |
| DONE    | any            | NEEDS_REVISION  | any             | NEEDS_REVISION           |
| DONE    | any            | any             | NEEDS_REVISION  | NEEDS_REVISION           |
| ERROR   | any            | any             | any             | ERROR                    |
| DONE    | ERROR          | any (not ERROR) | any (not ERROR) | Proceed with available   |
| DONE    | any            | ERROR           | any (not ERROR) | Proceed with available   |
| DONE    | any            | any (not ERROR) | ERROR           | Proceed with available   |
| any     | —              | —               | —               | 2+ missing/ERROR → ERROR |

**Additional V-aggregator logic that must migrate:**

- **Task ID failure mapping:** The planner's replan mode currently reads `verifier.md`'s Actionable Items section with specific task IDs. Without the aggregator, either: (a) the orchestrator must compile task ID failure mapping from V sub-agent memories and pass it to the planner, or (b) the planner must directly read `verification/v-tasks.md` for task-level failures. Option (b) is more aligned with the "read artifacts directly" philosophy.

**Migration requirement:** Orchestrator must read V sub-agent completion contracts and memories, apply the decision table above, and manage the Pattern C replan loop. The planner must be updated to read `verification/v-tasks.md` directly instead of `verifier.md` for replan mode.

#### R Cluster Completion (from r-aggregator → orchestrator)

**Current logic (r-aggregator.agent.md, Workflow step 9):**

1. R-Security ERROR or Blocker/Critical findings → `ERROR` (pipeline blocker)
2. R-Security missing → `ERROR` (mandatory)
3. 2+ non-knowledge sub-agents ERROR/missing → `ERROR`
4. ≥1 Major finding (no Blocker) → `NEEDS_REVISION`
5. All remaining → `DONE`
6. R-Knowledge status does NOT affect result

**Additional R-aggregator logic that must migrate:**

- R-Security pipeline override enforcement
- R-Knowledge non-blocking handling
- Extraction of high-value knowledge suggestions from `knowledge-suggestions.md`

**Migration requirement:** Orchestrator must: (1) check R-Security memory first for critical/blocker findings, (2) count available non-knowledge sub-agents, (3) check for Major findings across R-Quality and R-Testing memories, (4) ignore R-Knowledge errors. The orchestrator reads only memories, not full review artifacts.

#### Memory Write Migration

**Current:** Aggregators write to `memory.md` after merging sub-agent findings (they are the only cluster agents with write access).
**Proposed:** Orchestrator merges each sub-agent's isolated memory into `memory.md` after each cluster completes. This consolidates all memory writes into the orchestrator.

### Inter-Agent Dependency Graph

**Legend:** `A → B` means agent B depends on output from agent A.

```
orchestrator → [all agents] (initial-request.md, memory.md)

researcher(focused) ×4 → researcher(synthesis) → spec → designer → CT ×4 → ct-aggregator
                                                      ↘                         ↓
                                                    planner ←──────────────────┘
                                                      ↓
                                                implementer/doc-writer ×N
                                                      ↓
                                                  v-build → v-tests, v-tasks, v-feature → v-aggregator
                                                                                            ↓
                                                                              [replan loop: planner → impl → V cluster]
                                                                                            ↓
                                                                                    R ×4 → r-aggregator
```

**Critical dependency chains (longest sequential paths):**

1. researcher(synthesis) → spec → designer → CT cluster → ct-aggregator → planner (if NEEDS_REVISION also: → designer → CT cluster again)
2. planner → implementer waves → v-build → V parallel → v-aggregator → [replan loop ×3] → R cluster → r-aggregator

**Dependencies that change with aggregator removal:**

- `spec` no longer depends on researcher(synthesis); depends directly on researcher(focused) ×4 outputs
- `designer` no longer depends on researcher(synthesis); depends directly on researcher(focused) ×4 outputs
- `planner` no longer depends on researcher(synthesis); depends directly on researcher(focused) ×4 outputs
- `planner` no longer depends on ct-aggregator for `design_critical_review.md`; depends directly on `ct-review/ct-*.md` files
- `planner` (replan mode) no longer depends on v-aggregator for `verifier.md`; depends directly on `verification/v-*.md` files
- `designer` (revision mode) no longer depends on ct-aggregator for `design_critical_review.md`; depends directly on `ct-review/ct-*.md` files

### Integration Points

Key integration points between affected areas:

1. **Orchestrator ↔ Cluster sub-agents (new):** The orchestrator currently reads only aggregator completion contracts. After removal, it must read individual sub-agent completion contracts AND their isolated memory files.

2. **Orchestrator ↔ memory.md (expanded):** Currently aggregators and sequential agents write to memory.md. Post-change, the orchestrator must merge all isolated memory files after each parallelism boundary.

3. **Planner ↔ V sub-agents (new direct link):** The planner's replan mode currently reads `verifier.md`. It must shift to reading `verification/v-tasks.md` (for failing task IDs) and potentially `verification/v-tests.md` (for failing test details).

4. **Designer ↔ CT sub-agents (new direct link):** The designer's revision path currently reads `design_critical_review.md`. It must shift to reading all 4 `ct-review/ct-*.md` files.

5. **Spec/Designer/Planner ↔ Research files (widened):** These agents currently read 1 file (`analysis.md`). They must shift to reading 4 files (`research/*.md`).

6. **Dispatch patterns (updated):** Pattern A currently ends with "invoke aggregator." This step must change to "orchestrator reads sub-agent memories and applies completion contract." Pattern B similarly. Pattern C's replan loop references `verifier.md` which no longer exists.

7. **Memory write safety rules (updated):** The current read-write classification in Global Rule 12 of the orchestrator lists aggregators as read-write sequential agents. This categorization is removed. All agents become read-only for shared memory; only the orchestrator writes.

8. **NEEDS_REVISION routing table (updated):** The routing table currently routes through aggregators (e.g., "CT Aggregator → Designer", "V Aggregator → Planner"). These routes must change to "Orchestrator (reading CT memories) → Designer", "Orchestrator (reading V memories) → Planner".

## File References

| File                                                                               | Relevance                                                                                                                                                                            |
| ---------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| [orchestrator.agent.md](NewAgentsAndPrompts/orchestrator.agent.md)                 | Primary coordination logic; Global Rule 12 (memory write safety), cluster dispatch, NEEDS_REVISION routing table, memory lifecycle, parallel execution summary — all require updates |
| [ct-aggregator.agent.md](NewAgentsAndPrompts/ct-aggregator.agent.md)               | Being removed; completion contract logic (severity-based) must migrate to orchestrator                                                                                               |
| [v-aggregator.agent.md](NewAgentsAndPrompts/v-aggregator.agent.md)                 | Being removed; completion contract decision table and task ID failure mapping must migrate                                                                                           |
| [r-aggregator.agent.md](NewAgentsAndPrompts/r-aggregator.agent.md)                 | Being removed; R-Security pipeline override and complex completion rules must migrate                                                                                                |
| [researcher.agent.md](NewAgentsAndPrompts/researcher.agent.md)                     | Synthesis mode being removed; focused mode needs isolated memory output added                                                                                                        |
| [spec.agent.md](NewAgentsAndPrompts/spec.agent.md)                                 | Input changes: analysis.md → research/\*.md (4 files)                                                                                                                                |
| [designer.agent.md](NewAgentsAndPrompts/designer.agent.md)                         | Input changes: analysis.md → research/_.md; design_critical_review.md → ct-review/ct-_.md                                                                                            |
| [planner.agent.md](NewAgentsAndPrompts/planner.agent.md)                           | Input changes: analysis.md → research/_.md; design_critical_review.md → ct-review/ct-_.md; verifier.md → verification/v-\*.md                                                        |
| [dispatch-patterns.md](NewAgentsAndPrompts/dispatch-patterns.md)                   | Patterns A and B reference aggregator invocation step; must be updated                                                                                                               |
| [feature-workflow.prompt.md](NewAgentsAndPrompts/feature-workflow.prompt.md)       | High-level rules reference aggregators and analysis.md; must be updated                                                                                                              |
| [implementer.agent.md](NewAgentsAndPrompts/implementer.agent.md)                   | Needs isolated memory output added                                                                                                                                                   |
| [documentation-writer.agent.md](NewAgentsAndPrompts/documentation-writer.agent.md) | Needs isolated memory output added                                                                                                                                                   |
| [ct-security.agent.md](NewAgentsAndPrompts/ct-security.agent.md)                   | Needs isolated memory output added                                                                                                                                                   |
| [ct-scalability.agent.md](NewAgentsAndPrompts/ct-scalability.agent.md)             | Needs isolated memory output added                                                                                                                                                   |
| [ct-maintainability.agent.md](NewAgentsAndPrompts/ct-maintainability.agent.md)     | Needs isolated memory output added                                                                                                                                                   |
| [ct-strategy.agent.md](NewAgentsAndPrompts/ct-strategy.agent.md)                   | Needs isolated memory output added                                                                                                                                                   |
| [v-build.agent.md](NewAgentsAndPrompts/v-build.agent.md)                           | Needs isolated memory output added                                                                                                                                                   |
| [v-tests.agent.md](NewAgentsAndPrompts/v-tests.agent.md)                           | Needs isolated memory output added                                                                                                                                                   |
| [v-tasks.agent.md](NewAgentsAndPrompts/v-tasks.agent.md)                           | Needs isolated memory output added                                                                                                                                                   |
| [v-feature.agent.md](NewAgentsAndPrompts/v-feature.agent.md)                       | Needs isolated memory output added                                                                                                                                                   |
| [r-quality.agent.md](NewAgentsAndPrompts/r-quality.agent.md)                       | Needs isolated memory output added                                                                                                                                                   |
| [r-security.agent.md](NewAgentsAndPrompts/r-security.agent.md)                     | Needs isolated memory output added                                                                                                                                                   |
| [r-testing.agent.md](NewAgentsAndPrompts/r-testing.agent.md)                       | Needs isolated memory output added                                                                                                                                                   |
| [r-knowledge.agent.md](NewAgentsAndPrompts/r-knowledge.agent.md)                   | Needs isolated memory output added; already has complex outputs                                                                                                                      |

## Assumptions & Limitations

1. **Assumed isolated memory format:** The research assumes isolated memory files will be compact summaries (severity findings, completion status, key decisions) that the orchestrator can efficiently parse. The exact format is not specified in the initial request and must be designed.
2. **Assumed orchestrator capacity:** The orchestrator is assumed to be capable of absorbing all aggregator completion contract logic without exceeding context window limits. The combined logic from 3 aggregators (CT, V, R) plus researcher synthesis is substantial.
3. **Assumed planner can read V sub-agent files directly:** The planner's replan dependency on verifier.md's Actionable Items section is assumed resolvable by having the planner read `verification/v-tasks.md` directly for failing task IDs. However, `verification/v-tests.md` cross-references (test failures mapped to task IDs) currently live only in the aggregator's merge step and may need to be supplied differently.
4. **No existing decisions.md found** — no prior architectural decisions to incorporate.
5. **Scope limited to prompt files:** This analysis covers only the agent definition files in `NewAgentsAndPrompts/`. Any runtime infrastructure, agent dispatch framework, or memory management code outside this directory is not examined.

## Open Questions

1. **Isolated memory file naming convention:** What is the naming pattern for isolated memory files? E.g., `memory-<agent-name>.md`? Where do they live — in the feature directory root or a subdirectory?
2. **Orchestrator memory merge strategy:** How does the orchestrator merge isolated memories into `memory.md`? Simple concatenation? Selective extraction? What conflict resolution applies?
3. **Task ID failure mapping without V-Aggregator:** The planner's replan mode depends on a compiled task-ID-to-failure mapping currently produced by v-aggregator. Should the planner compile this itself from raw V sub-agent files, or should the orchestrator compile it and pass it in?
4. **Unresolved tensions detection:** Aggregators currently detect contradictions between sub-agent findings and surface them as "Unresolved Tensions." Who performs this cross-checking in the new architecture? The orchestrator (from memories only) or the downstream consumer (from full artifacts)?
5. **CT severity assessment without aggregator:** The CT completion contract requires assessing findings across all 4 sub-agents to find the highest severity. Can the orchestrator reliably do this from memory summaries alone, or does each CT sub-agent need to include its highest-severity finding explicitly in its memory?
6. **R-Security pipeline override from memory:** The R-Security override requires checking severity of specific findings. Must R-Security's isolated memory include an explicit "highest severity: Critical/Blocker" field for the orchestrator to read?
7. **Deduplication loss:** Aggregators currently deduplicate findings across sub-agents. Without aggregators, duplicates remain in individual files. Is this acceptable, or should downstream consumers handle deduplication?

## Research Metadata

- confidence_level: high
- coverage_estimate: All 25 agent/prompt files in NewAgentsAndPrompts/ were examined. Full input/output contracts, completion contracts, memory write rules, and inter-agent dependencies were mapped for every agent.
- gaps: (1) No runtime code or agent dispatch framework was examined — only prompt definitions. If there is additional orchestration logic outside these files, it was not covered. (2) The exact mechanism for how `runSubagent` passes inputs and reads outputs at runtime was not examined — only the declared contracts in prompt files. (3) The deprecated `critical-thinker.agent.md` was read but not deeply analyzed since it is already superseded.
