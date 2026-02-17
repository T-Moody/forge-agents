# Architecture Research — arch-upgrade-remaining-concerns

## Focus Area

Architecture

## Summary

The orchestrator is 489 lines (exceeds its own 450-line target) with the Workflow Steps section consuming ~245 lines alone. The 200-line memory emergency prune rule appears in exactly 2 locations. All 22 agent files use only `name` and `description` in YAML frontmatter — no `memory_access` field exists anywhere. The Cluster Dispatch Patterns section spans ~40 lines and is a candidate for extraction. R-Knowledge receives the widest input set (10 sources) among all sub-agents.

---

## Findings

### 1. Memory System Architecture

**Global Rule 6** (orchestrator.agent.md, line 34):

> "Memory-First Protocol: Initialize `memory.md` at Step 0. Prune memory at pipeline checkpoints (after Steps 1.2, 2, 4). Invalidate memory entries on step failure/revision. Emergency prune if `memory.md` exceeds 200 lines (keep only Lessons Learned + Artifact Index + current-phase entries). Memory failure is non-blocking — if `memory.md` cannot be created or becomes corrupted, log a warning and proceed. Agents fall back to direct artifact reads."

**Memory Lifecycle Actions table** (orchestrator.agent.md, lines 407–419):
The table has 7 rows:

| Action                 | When                                       | What                                                                           |
| ---------------------- | ------------------------------------------ | ------------------------------------------------------------------------------ |
| Initialize             | Step 0                                     | Create memory.md with empty template                                           |
| Prune                  | After Steps 1.2, 2, 4                      | Remove entries older than 2 completed phases. Preserve Lessons Learned.        |
| Extract Lessons        | Between implementation waves               | Read completed task outputs; append to Lessons Learned                         |
| Invalidate on revision | Before dispatching revision agent          | Mark affected entries with `[INVALIDATED — <reason>]`                          |
| Clean invalidated      | After revision agent completes             | Remove remaining `[INVALIDATED]` entries not replaced                          |
| **Emergency prune**    | **When memory exceeds 200 lines**          | **Remove all except Lessons Learned + Artifact Index + current-phase entries** |
| Validate               | After aggregators/sequential agents return | Check that agent wrote to memory. Log warning if not.                          |

**Anti-Drift Anchor** (orchestrator.agent.md, line 485) also references emergency prune:

> "You manage the memory lifecycle (init, prune, invalidate, emergency prune)."

**The 200-line emergency prune rule appears in exactly 3 locations:**

1. Global Rule 6 (line 34) — policy rule
2. Memory Lifecycle Actions table, row 6 (line 416) — operational table
3. Anti-Drift Anchor (line 485) — reinforcement reminder

**No other agent files reference the 200-line limit for memory.** The only other occurrences of "200" across agent files are the `read_file` limit of ~200 lines per call in Operating Rules — unrelated to memory pruning.

---

### 2. Dispatch Pattern Architecture

**Section location:** Cluster Dispatch Patterns, lines 82–120 (~40 lines).

**Pattern A — Fully Parallel** (lines 86–91, ~6 steps):
Used by: Research cluster (Step 1), CT cluster (Step 3b), R cluster (Step 7).
Steps: dispatch N parallel → wait → retry errors → aggregator if ≥2 outputs → check contract.

**Pattern B — Sequential Gate + Parallel** (lines 93–99, ~7 steps):
Used by: V cluster (Step 6), wrapped inside Pattern C.
Steps: dispatch gate → wait → if gate ERROR retry → if DONE dispatch N-1 parallel → wait → aggregator → check contract.

**Pattern C — Replan Loop** (lines 101–114, ~14 lines including code block):
Wraps Pattern B for V cluster. Max 3 iterations. On NEEDS_REVISION or ERROR: invalidate memory → invoke planner in replan mode → execute fix tasks → re-run Pattern B.

**Usage in Workflow Steps (where patterns are re-described):**

- Step 1 (Research, lines 155–184): Re-describes Pattern A application (~30 lines)
- Step 3b (CT Cluster, lines 201–238): Re-describes Pattern A application (~38 lines)
- Step 6 (V Cluster, lines 275–318): Re-describes Pattern B+C application (~44 lines)
- Step 7 (R Cluster, lines 320–365): Re-describes Pattern A application (~46 lines)

**Total lines dedicated to pattern definitions + pattern usage across workflow steps:** ~40 (definitions) + ~158 (usage in steps) = ~198 lines. The step-level descriptions repeat substantial detail from the pattern definitions, particularly error handling and concurrency rules.

**Extraction potential:** The pattern definitions (lines 82–120) are already separated but short (~40 lines). The real bulk is the repeated application in Steps 1, 3b, 6, and 7 (~158 lines). Extracting the patterns themselves would save only ~40 lines from the orchestrator, but if the step-level descriptions are simplified to reference extracted patterns, significant reduction is possible.

---

### 3. Agent YAML Frontmatter Conventions

**Every active agent file** uses exactly 2 YAML fields:

| Field         | Description                                            | Present in         |
| ------------- | ------------------------------------------------------ | ------------------ |
| `name`        | Agent identifier (e.g., `orchestrator`, `ct-security`) | All 22 agent files |
| `description` | One-line agent purpose description                     | All 22 agent files |

**Full inventory of YAML frontmatter across all agent files:**

| Agent File                    | `name`               | `description` value (shortened)                           |
| ----------------------------- | -------------------- | --------------------------------------------------------- |
| orchestrator.agent.md         | orchestrator         | Deterministic workflow orchestrator...                    |
| researcher.agent.md           | researcher           | Investigates existing codebase...                         |
| spec.agent.md                 | spec                 | Produces a clear, testable feature specification...       |
| designer.agent.md             | designer             | Creates a technical design document...                    |
| implementer.agent.md          | implementer          | Implements exactly one task using TDD...                  |
| documentation-writer.agent.md | documentation-writer | Generates and maintains project documentation...          |
| planner.agent.md              | planner              | Creates dependency-aware implementation plans...          |
| ct-security.agent.md          | ct-security          | Security-focused critical thinking sub-agent...           |
| ct-scalability.agent.md       | ct-scalability       | Scalability-focused critical thinking sub-agent...        |
| ct-maintainability.agent.md   | ct-maintainability   | Critical thinking sub-agent focused on maintainability... |
| ct-strategy.agent.md          | ct-strategy          | Critical thinking sub-agent focused on strategic risks... |
| ct-aggregator.agent.md        | ct-aggregator        | Aggregates 4 CT sub-agent findings...                     |
| v-build.agent.md              | v-build              | Build system detection and compilation gate...            |
| v-tests.agent.md              | v-tests              | Full test suite execution and analysis...                 |
| v-tasks.agent.md              | v-tasks              | Per-task acceptance criteria verification...              |
| v-feature.agent.md            | v-feature            | Feature-level acceptance criteria verification...         |
| v-aggregator.agent.md         | v-aggregator         | Aggregates Verification cluster results...                |
| r-quality.agent.md            | r-quality            | Code quality, readability, and maintainability review...  |
| r-security.agent.md           | r-security           | Security scanning and OWASP compliance review...          |
| r-testing.agent.md            | r-testing            | Test quality and coverage adequacy review...              |
| r-knowledge.agent.md          | r-knowledge          | Knowledge evolution agent...                              |
| r-aggregator.agent.md         | r-aggregator         | Aggregates Review cluster findings...                     |

**No `memory_access` field exists** in any agent file. Memory read/write rules are enforced only via prose instructions in:

- Orchestrator workflow steps (e.g., "Sub-agents read memory but do NOT write to it")
- Individual agent body text (not frontmatter)

**Deprecated agent:** `critical-thinker.agent.md` has an unusual structure — a deprecation notice appears before the `---` YAML delimiter (lines 1–5), making its frontmatter malformed. Its `---` and YAML fields appear at lines 7–9.

**Prompt file:** `feature-workflow.prompt.md` uses different YAML fields: `name` ("Feature Workflow") and `agent` ("orchestrator") — not the same schema as agent files (no `description` field).

---

### 4. Orchestrator Line Count & Section Breakdown

**Total lines: 489** (exceeds the stated 450-line target from the initial request)

| Section                          | Lines   | Line Range  | % of File |
| -------------------------------- | ------- | ----------- | --------- |
| YAML frontmatter                 | 5       | 1–5         | 1.0%      |
| Title + intro                    | 11      | 6–16        | 2.2%      |
| Inputs                           | 4       | 17–20       | 0.8%      |
| Outputs                          | 6       | 21–26       | 1.2%      |
| **Global Rules**                 | **14**  | **27–40**   | **2.9%**  |
| **Documentation Structure**      | **28**  | **41–68**   | **5.7%**  |
| **Operating Rules**              | **13**  | **69–81**   | **2.7%**  |
| **Cluster Dispatch Patterns**    | **40**  | **82–121**  | **8.2%**  |
| **Workflow Steps**               | **245** | **122–366** | **50.1%** |
| **NEEDS_REVISION Routing Table** | **13**  | **368–380** | **2.7%**  |
| **Expectations Per Agent**       | **26**  | **381–406** | **5.3%**  |
| **Memory Lifecycle Actions**     | **14**  | **407–420** | **2.9%**  |
| **Parallel Execution Summary**   | **53**  | **421–473** | **10.8%** |
| Completion Contract              | 9       | 474–482     | 1.8%      |
| Anti-Drift Anchor                | 7       | 483–489     | 1.4%      |

**Largest sections:**

1. **Workflow Steps: 245 lines (50.1%)** — by far the largest; contains the full 8-step pipeline including all cluster dispatch detail
2. **Parallel Execution Summary: 53 lines (10.8%)** — ASCII diagram summarizing the pipeline visually; partially redundant with Workflow Steps
3. **Cluster Dispatch Patterns: 40 lines (8.2%)** — defines Patterns A, B, C abstractly
4. **Documentation Structure: 28 lines (5.7%)** — lists all artifact paths
5. **Expectations Per Agent: 26 lines (5.3%)** — table mapping agents to expected behavior

**Workflow Steps sub-breakdown (245 lines):**

| Sub-section                     | Lines | Line Range |
| ------------------------------- | ----- | ---------- |
| Step 0: Setup                   | 29    | 126–154    |
| Step 1: Research (Pattern A)    | 30    | 155–184    |
| Step 2: Specification           | 8     | 186–193    |
| Step 3: Design                  | 7     | 194–200    |
| Step 3b: CT Cluster (Pattern A) | 39    | 201–239    |
| Step 4: Planning                | 11    | 240–250    |
| Step 5: Implementation          | 23    | 252–274    |
| Step 6: V Cluster (Pattern B+C) | 45    | 275–319    |
| Step 7: R Cluster (Pattern A)   | 47    | 320–366    |

The cluster dispatch steps (1, 3b, 6, 7) together consume 161 lines, which is 65.7% of the Workflow Steps section and 32.9% of the entire file.

---

### 5. R-Knowledge Input Architecture

**File:** r-knowledge.agent.md (283 lines total)

**Inputs section (lines 22–30)** lists 10 input sources:

1. `memory.md` — read first (operational memory)
2. `initial-request.md`
3. `feature.md`
4. `design.md`
5. `plan.md`
6. `verifier.md`
7. `decisions.md` (if present)
8. `.github/instructions/` (if present)
9. Git diff
10. Entire codebase

**Operating Rule 6 (line 42):** "Memory-first reading: Read `memory.md` FIRST before accessing any artifact. Use the Artifact Index to navigate directly to relevant sections rather than reading full artifacts."

**Workflow Step 2 (lines 106–108)** — "Read Pipeline Artifacts":

> "Read `initial-request.md`, `feature.md`, `design.md`, `plan.md`, and `verifier.md` for full pipeline history and context. Use the Artifact Index from memory to navigate directly to relevant sections."

This step instructs reading all 5 pipeline artifacts in full, with the Artifact Index as an optimization hint — not as a replacement for full reads.

**Orchestrator dispatch (orchestrator.agent.md, line 335):**
The R cluster dispatch table passes to r-knowledge: `tier, initial-request.md, feature.md, design.md, plan.md, verifier.md`

**Comparison with other R sub-agents:**

- r-quality: `tier, initial-request.md, git diff context` (3 inputs)
- r-security: `tier, initial-request.md, git diff context` (3 inputs)
- r-testing: `tier, initial-request.md, git diff context` (3 inputs)
- **r-knowledge: `tier, initial-request.md, feature.md, design.md, plan.md, verifier.md` (6 inputs)** — 2× the inputs of other R sub-agents

R-Knowledge also reads `decisions.md`, `.github/instructions/`, git diff, and "entire codebase" per its own Inputs section — but these are self-directed reads, not orchestrator-provided inputs.

---

## File References

| File                                                        | Relevance                                                              |
| ----------------------------------------------------------- | ---------------------------------------------------------------------- |
| `NewAgentsAndPrompts/orchestrator.agent.md` (489 lines)     | Primary file for memory system, dispatch patterns, line count analysis |
| `NewAgentsAndPrompts/r-knowledge.agent.md` (283 lines)      | Input architecture analysis                                            |
| All 22 `*.agent.md` files in `NewAgentsAndPrompts/`         | YAML frontmatter survey                                                |
| `NewAgentsAndPrompts/feature-workflow.prompt.md` (52 lines) | Prompt file frontmatter comparison                                     |
| `NewAgentsAndPrompts/critical-thinker.agent.md` (124 lines) | Deprecated; malformed frontmatter                                      |

---

## Assumptions & Limitations

- **Line counts** are based on the file state at time of research. If files have been edited concurrently, counts may be stale.
- **No `decisions.md`** exists anywhere in the repository — confirmed via file search.
- The deprecated `critical-thinker.agent.md` was included in the frontmatter survey for completeness but is not part of the active agent set.
- The analysis does not cover how VS Code / Copilot runtime parses YAML frontmatter fields — only what fields currently exist in the files.

---

## Open Questions

1. **YAML frontmatter field validation:** Does the VS Code Copilot agent runtime validate or use fields beyond `name` and `description`? If it ignores unknown fields, adding `memory_access` is safe. If it validates strictly, unknown fields could cause agent load failures.
2. **Pattern extraction target:** Would extracting patterns to a separate file require all agents that reference patterns to also receive that file as input, or is it only the orchestrator's own reference?
3. **Parallel Execution Summary redundancy:** The 53-line ASCII summary (lines 421–473) substantially overlaps with the Workflow Steps section. Is it intended as a quick-reference for the AI, or could it be removed to save lines?

---

## Research Metadata

- **confidence_level:** high — all 22 agent files were read; all relevant orchestrator sections were examined with exact line numbers.
- **coverage_estimate:** Complete coverage of all `.agent.md` and `.prompt.md` files in `NewAgentsAndPrompts/`. Full read of `orchestrator.agent.md` (489/489 lines) and `r-knowledge.agent.md` (283/283 lines).
- **gaps:** No gaps identified for this focus area. The `critical-thinker.agent.md` deprecation notice/malformed frontmatter was noted but does not affect the active agent set.
