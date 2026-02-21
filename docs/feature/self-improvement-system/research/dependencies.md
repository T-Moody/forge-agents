# Research: Dependencies

## Focus Area

**dependencies** — Module interactions, data flow, API/interface contracts, external dependencies, integration points between affected areas.

## Summary

The Forge pipeline has a strict, linear artifact consumption chain with well-defined I/O contracts per agent; the self-improvement system must hook into the orchestrator's dispatch/merge loop, extend each consuming agent's output contract, and introduce new storage directories and a post-mortem agent without altering existing data flow.

## Findings

### 1. Artifact Consumption Chain

The pipeline follows a strict sequential-then-parallel artifact consumption pattern. Each agent reads upstream artifacts (via memory-first protocol) and produces downstream artifacts. The full chain:

```
initial-request.md
  → researcher ×4 (parallel, Pattern A)
    → research/architecture.md, research/impact.md, research/dependencies.md, research/patterns.md
      → spec → feature.md
        → designer → design.md
          → CT cluster ×4 (parallel, Pattern A) → ct-review/ct-*.md
            → planner → plan.md + tasks/*.md
              → implementer/documentation-writer ×N (parallel waves, ≤4)
                → V cluster (Pattern B+C) → verification/v-*.md
                  → R cluster ×4 (parallel, Pattern A) → review/r-*.md
```

**Which agents consume which artifacts (evaluation targets):**

| Consuming Agent      | Reads (Source Artifacts)                                                                                         | Evaluation Target(s)                                        |
| -------------------- | ---------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------- |
| spec                 | research/architecture.md, research/impact.md, research/dependencies.md, research/patterns.md, initial-request.md | All 4 research/\*.md files                                  |
| designer             | feature.md, research/\*.md (selective)                                                                           | feature.md (from spec)                                      |
| ct-security          | design.md, feature.md                                                                                            | design.md (from designer)                                   |
| ct-scalability       | design.md, feature.md                                                                                            | design.md (from designer)                                   |
| ct-maintainability   | design.md, feature.md                                                                                            | design.md (from designer)                                   |
| ct-strategy          | design.md, feature.md                                                                                            | design.md (from designer)                                   |
| planner              | design.md, feature.md, research/_.md, ct-review/ct-_.md (conditional)                                            | design.md (from designer)                                   |
| implementer          | tasks/\<task\>.md, feature.md, design.md                                                                         | tasks/\<task\>.md (from planner), design.md (from designer) |
| documentation-writer | tasks/\<task\>.md, feature.md, design.md                                                                         | tasks/\<task\>.md (from planner)                            |
| v-build              | feature.md, design.md, plan.md, tasks/\*.md                                                                      | (Evaluates codebase, not upstream artifacts)                |
| v-tests              | verification/v-build.md                                                                                          | v-build.md (from v-build)                                   |
| v-tasks              | verification/v-build.md, plan.md, tasks/\*.md                                                                    | v-build.md (from v-build)                                   |
| v-feature            | verification/v-build.md, feature.md, design.md                                                                   | v-build.md (from v-build), feature.md                       |
| r-quality            | design.md, git diff, codebase                                                                                    | (Evaluates code, not upstream artifacts directly)           |
| r-security           | initial-request.md, git diff, codebase                                                                           | (Evaluates code)                                            |
| r-testing            | initial-request.md, feature.md, git diff, codebase                                                               | feature.md                                                  |
| r-knowledge          | initial-request.md, decisions.md, codebase                                                                       | (Meta-analysis across pipeline artifacts)                   |

### 2. Memory File Relationships (Isolated → Shared Merge Pattern)

**Data flow pattern:**

1. Each agent writes to `memory/<agent-name>.mem.md` (isolated)
2. Orchestrator reads isolated memory after each agent/cluster completes
3. Orchestrator merges Key Findings, Decisions, and Artifact Index into shared `memory.md`
4. Downstream agents read shared `memory.md` first, then upstream isolated memories for targeted reads

**Memory file naming convention:**

- Researchers: `memory/researcher-<focus-area>.mem.md`
- CT agents: `memory/ct-<focus>.mem.md`
- V agents: `memory/v-<focus>.mem.md`
- R agents: `memory/r-<focus>.mem.md`
- Implementers: `memory/implementer-<task-id>.mem.md`
- Documentation writers: `memory/documentation-writer-<task-id>.mem.md`

**Isolated memory structure (standard across all agents):**

```markdown
# Memory: <agent-name>

## Status

<DONE|NEEDS_REVISION|ERROR>: <one-line summary>

## Key Findings

- <finding 1> ... (≤5 bullets)

## Highest Severity

<value per cluster taxonomy>
## Decisions Made
- <decision> (optional)
## Artifact Index
- <artifact-path> — §<Section> (brief relevance note)
```

**Memory lifecycle actions (orchestrator-managed):**

- Initialize (Step 0)
- Merge (after each agent/cluster)
- Prune (after Steps 1.1m, 2m, 4m)
- Extract Lessons (between implementation waves)
- Invalidate on revision
- Clean invalidated (after revision completes)
- Validate (check agent wrote isolated memory)

### 3. Orchestrator Dispatch Patterns and Telemetry Integration Points

The orchestrator dispatches agents via `runSubagent` calls. There is **no programmatic runtime** — the entire system is prompt-driven markdown agents executed by the LLM runtime (GitHub Copilot). Therefore:

**Current dispatch flow (where telemetry must be captured):**

| Step | Dispatch Method                    | Pattern    | Agents                      | Integration Point for Telemetry      |
| ---- | ---------------------------------- | ---------- | --------------------------- | ------------------------------------ |
| 1.1  | 4× `runSubagent` (parallel)        | Pattern A  | researcher ×4               | Before/after each `runSubagent` call |
| 2    | 1× `runSubagent` (sequential)      | Sequential | spec                        | Before/after single call             |
| 3    | 1× `runSubagent` (sequential)      | Sequential | designer                    | Before/after single call             |
| 3b   | 4× `runSubagent` (parallel)        | Pattern A  | CT ×4                       | Before/after each call               |
| 4    | 1× `runSubagent` (sequential)      | Sequential | planner                     | Before/after single call             |
| 5    | N× `runSubagent` (parallel waves)  | Wave-based | implementer/doc-writer ×N   | Before/after each call per sub-wave  |
| 6.1  | 1× `runSubagent` (sequential gate) | Pattern B  | v-build                     | Before/after gate call               |
| 6.2  | 3× `runSubagent` (parallel)        | Pattern B  | v-tests, v-tasks, v-feature | Before/after each call               |
| 7    | 4× `runSubagent` (parallel)        | Pattern A  | R ×4                        | Before/after each call               |

**Key constraint:** The orchestrator is prompt-driven, not programmatic code. Telemetry "capture" means the orchestrator writes structured metadata to files, not calling APIs. The `runSubagent` call returns a completion contract string (`DONE:`, `NEEDS_REVISION:`, `ERROR:`), and the orchestrator then reads the agent's isolated memory file. Telemetry must be captured through these natural observation points.

**Implications for telemetry tracking (PART 2 of initial request):**

- `agent_name`: Known at dispatch time
- `execution_time_ms`: **Not directly available** — the orchestrator does not have a timer. It could log start/end timestamps if the runtime provides a clock, but this is platform-dependent.
- `retry_count`: Observable — orchestrator tracks retries per Global Rule 4
- `failure_reason`: Available from completion contract string and memory files
- `number_of_iterations`: Observable — Pattern C loop counter
- `human_intervention_required`: Observable — from APPROVAL_MODE and NEEDS_REVISION escalations

### 4. Integration Points for Artifact Evaluations (PART 1)

**Where evaluations would be produced:** Every agent that CONSUMES an upstream artifact (see §1 table above) would append or co-produce an `artifact_evaluation` section.

**Agent output contract changes required:**

| Agent                | Current Outputs                               | New Additional Output                                   |
| -------------------- | --------------------------------------------- | ------------------------------------------------------- |
| spec                 | feature.md + memory/spec.mem.md               | Evaluation of research/\*.md files → evaluation file(s) |
| designer             | design.md + memory/designer.mem.md            | Evaluation of feature.md → evaluation file              |
| CT sub-agents ×4     | ct-review/ct-_.md + memory/ct-_.mem.md        | Evaluation of design.md → evaluation file               |
| planner              | plan.md + tasks/\*.md + memory/planner.mem.md | Evaluation of design.md → evaluation file               |
| implementer          | code + tests + memory/implementer-\*.mem.md   | Evaluation of tasks/\*.md + design.md → evaluation file |
| documentation-writer | docs + memory/documentation-writer-\*.mem.md  | Evaluation of tasks/\*.md → evaluation file             |
| v-tests              | verification/v-tests.md + memory              | Evaluation of v-build.md → evaluation file              |
| v-tasks              | verification/v-tasks.md + memory              | Evaluation of v-build.md, tasks/\*.md → evaluation file |
| v-feature            | verification/v-feature.md + memory            | Evaluation of v-build.md, feature.md → evaluation file  |
| r-testing            | review/r-testing.md + memory                  | Evaluation of feature.md → evaluation file              |

**Storage options for evaluations:**

- Option A: Append as YAML section to each agent's primary output artifact
- Option B: Separate file per evaluation in `docs/feature/<slug>/artifact-evaluations/`
- Option C: Append to the agent's isolated memory file

The initial request specifies: "Appended to agent output" OR "stored in a dedicated evaluation directory." Both are compatible with the current architecture.

### 5. Data Flow for Post-Mortem Agent (PART 3)

**Proposed data flow:**

```
All agents produce:
  → artifact evaluations (per §4)
  → isolated memory files (existing)

Orchestrator captures:
  → telemetry/run-log.yaml (per §3)

Post-mortem agent reads:
  ← artifact-evaluations/*.yaml (or *.md)
  ← telemetry/run-log.yaml
  ← memory/*.mem.md (all isolated memories)

Post-mortem agent writes:
  → post-mortems/<timestamp>-post-mortem.md (or .yaml)
```

**Pipeline integration point:** The PostMortemAgent would run AFTER Step 7 (R cluster) completes — as a new Step 8. It must be additive (non-blocking) — pipeline success/failure is decided at Step 7, and the post-mortem runs regardless.

**Relationship to r-knowledge:** The r-knowledge agent already performs similar meta-analysis — it examines pipeline artifacts and produces `knowledge-suggestions.md` and `decisions.md`. Key differences:

- r-knowledge produces _forward-looking suggestions_ for human review (what to change in agent definitions)
- Post-mortem produces _backward-looking analysis_ of the run that just happened (what went well/poorly)
- r-knowledge runs as part of the R cluster (Step 7) and is non-blocking
- Post-mortem would run after the R cluster (Step 8) and is also non-blocking
- r-knowledge has access to the codebase and git diff; post-mortem would focus on pipeline metadata and evaluations
- There is potential overlap — both analyze pipeline behavior. The design should clarify the boundary.

### 6. Storage Directory Dependencies (PART 4)

**Existing directory structure under `docs/feature/<feature-slug>/`:**

```
initial-request.md
memory.md
memory/           (agent-isolated memories)
research/         (researcher outputs)
ct-review/        (CT cluster outputs)
verification/     (V cluster outputs)
review/           (R cluster outputs)
tasks/            (planner task files)
```

**New directories required:**

```
agent-metrics/         (telemetry run logs — PART 2)
artifact-evaluations/  (structured evaluations — PART 1)
post-mortems/          (post-mortem reports — PART 3)
```

**Question: Feature-scoped or global?**
The initial request says `/agent-metrics/`, `/artifact-evaluations/`, `/post-mortems/` — using root-like paths. But the current artifact structure is feature-scoped (`docs/feature/<slug>/`). The feature request doesn't specify whether storage is per-feature or cross-feature. This is a design decision — per-feature storage aligns with existing conventions; cross-feature storage enables cross-run pattern analysis.

### 7. Orchestrator Tool Restriction (Additional Requirement)

The initial request requires restricting the orchestrator to only `[agent, agent/runSubagent, memory]` tools. Currently, the orchestrator's instructions reference file operations:

- Step 0: "Create `initial-request.md`" and "Initialize `memory.md`" — these require file write tools
- Memory merge operations: the orchestrator directly writes to `memory.md`
- Memory pruning, invalidation, cleaning: all require file write tools

**Impact on telemetry tracking:** If the orchestrator loses file write access, it cannot directly write telemetry data. This means either:

1. A dedicated telemetry-writing subagent must be introduced
2. Or the memory tool must be extended to handle telemetry writes
3. Or the setup subagent (already needed) handles initial file creation while the orchestrator uses the `memory` tool for ongoing writes

This interacts heavily with PART 2 (telemetry) — the orchestrator observes dispatch events but may not be able to persist them directly.

### 8. External Dependencies

**No external code dependencies exist.** The entire system is markdown-based agent definitions processed by GitHub Copilot's agent runtime. There are:

- No npm packages or runtime dependencies
- No build system
- No external APIs
- No databases

The only "dependencies" are:

1. **GitHub Copilot agent runtime** — provides `runSubagent`, `memory`, file tools, `semantic_search`, `grep_search`, `read_file`, etc.
2. **File system** — all persistence is via markdown/YAML files
3. **Git** — diff context for R cluster agents

### 9. Cross-Agent Interface Contracts

**Completion contract (universal):**
All agents return exactly one of:

- `DONE: <summary>`
- `NEEDS_REVISION: <summary>` (where applicable)
- `ERROR: <reason>`

**Memory file contract (universal):**
All agents produce `memory/<agent-name>.mem.md` with the standard structure (§2).

**Evaluation contract (new, needed):**
A new universal contract section must be defined for artifact evaluations. Per the initial request, this would be a YAML block appended to agent output or written to a dedicated file:

```yaml
artifact_evaluation:
  source_artifact: <path>
  usefulness_score: (1-10)
  clarity_score: (1-10)
  useful_elements: [...]
  missing_information: [...]
  information_not_used: [...]
  inaccuracies: [...]
  impact_on_work: [...]
```

This evaluation contract must be added to each consuming agent's "Outputs" section and agent instructions — a cross-cutting concern affecting 10+ agent definition files.

## File References

| File/Folder                                                                                                        | Relevance                                                                                          |
| ------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------- |
| [.github/agents/orchestrator.agent.md](.github/agents/orchestrator.agent.md)                                       | Central dispatch logic, memory merge, cluster decision flows, all integration points for telemetry |
| [.github/agents/dispatch-patterns.md](.github/agents/dispatch-patterns.md)                                         | Pattern A/B/C definitions affecting how telemetry capture wraps dispatch                           |
| [.github/agents/researcher.agent.md](.github/agents/researcher.agent.md)                                           | Produces research artifacts consumed by spec (evaluation source)                                   |
| [.github/agents/spec.agent.md](.github/agents/spec.agent.md)                                                       | Consumes research→produces feature.md; first evaluation producer                                   |
| [.github/agents/designer.agent.md](.github/agents/designer.agent.md)                                               | Consumes feature.md→produces design.md; evaluation producer                                        |
| [.github/agents/planner.agent.md](.github/agents/planner.agent.md)                                                 | Consumes design.md→produces plan.md+tasks; evaluation producer                                     |
| [.github/agents/implementer.agent.md](.github/agents/implementer.agent.md)                                         | Consumes tasks→produces code; evaluation producer                                                  |
| [.github/agents/ct-security.agent.md](.github/agents/ct-security.agent.md)                                         | CT sub-agent; consumes design.md; evaluation producer                                              |
| [.github/agents/v-build.agent.md](.github/agents/v-build.agent.md)                                                 | V gate agent; consumed by v-tests, v-tasks, v-feature                                              |
| [.github/agents/v-tests.agent.md](.github/agents/v-tests.agent.md)                                                 | Consumes v-build.md; evaluation producer                                                           |
| [.github/agents/v-tasks.agent.md](.github/agents/v-tasks.agent.md)                                                 | Consumes v-build.md, tasks; evaluation producer                                                    |
| [.github/agents/v-feature.agent.md](.github/agents/v-feature.agent.md)                                             | Consumes v-build.md, feature.md; evaluation producer                                               |
| [.github/agents/r-knowledge.agent.md](.github/agents/r-knowledge.agent.md)                                         | Overlap with post-mortem role; produces knowledge-suggestions.md and decisions.md                  |
| [.github/agents/r-quality.agent.md](.github/agents/r-quality.agent.md)                                             | R sub-agent; evaluates code quality                                                                |
| [.github/agents/r-security.agent.md](.github/agents/r-security.agent.md)                                           | R sub-agent; security is pipeline-blocking                                                         |
| [.github/agents/r-testing.agent.md](.github/agents/r-testing.agent.md)                                             | R sub-agent; consumes feature.md; evaluation producer                                              |
| [.github/agents/documentation-writer.agent.md](.github/agents/documentation-writer.agent.md)                       | Consumes tasks; evaluation producer                                                                |
| [.github/prompts/feature-workflow.prompt.md](.github/prompts/feature-workflow.prompt.md)                           | Entry point prompt; may need new variables for self-improvement features                           |
| [docs/feature/self-improvement-system/initial-request.md](docs/feature/self-improvement-system/initial-request.md) | Feature requirements (6 parts + orchestrator tool restriction)                                     |

## Assumptions & Limitations

1. **Assumed:** The GitHub Copilot agent runtime provides `runSubagent` as the sole dispatch mechanism. There are no hooks, events, or middleware for instrumentation — all telemetry must be instruction-driven.
2. **Assumed:** The `memory` tool referenced in the orchestrator tool restriction is a runtime-provided tool for reading/writing memory files, not a new tool to be built.
3. **Assumed:** Timestamps are available to the orchestrator via the LLM runtime (e.g., `{{CURRENT_DATE}}`). If not, timestamp-based telemetry would degrade.
4. **Limitation:** I did not find any existing telemetry, metrics, or evaluation infrastructure in the codebase — this is entirely new capability.
5. **Limitation:** The README references aggregator agents (ct-aggregator, v-aggregator, r-aggregator) that do not exist as `.agent.md` files — the orchestrator now handles aggregation directly via cluster decision flows. The README is partially outdated.

## Open Questions

1. **Per-feature vs. global storage:** Should `agent-metrics/`, `artifact-evaluations/`, and `post-mortems/` be scoped under `docs/feature/<slug>/` (per-feature, aligns with existing conventions) or under a global `docs/` path (enables cross-feature analysis)?
2. **Orchestrator tool restriction vs. telemetry writing:** If the orchestrator is restricted to `[agent, agent/runSubagent, memory]`, who writes telemetry logs? A dedicated setup/telemetry subagent? The `memory` tool?
3. **Post-mortem vs. r-knowledge boundary:** Both analyze pipeline behavior. Should post-mortem subsume r-knowledge's analysis, complement it, or be entirely separate?
4. **Evaluation granularity:** Some agents consume multiple artifacts (e.g., spec reads 4 research files). Should the agent produce one evaluation per consumed artifact, or a single combined evaluation?
5. **Execution time tracking:** The orchestrator is an LLM agent without direct timer access. How should execution duration be captured? Platform-dependent? Estimated by the agent? Omitted?

## Research Metadata

- **confidence_level:** high — all 20 agent definition files, dispatch patterns, and the orchestrator workflow were read and analyzed
- **coverage_estimate:** Complete coverage of all `.agent.md` files, `dispatch-patterns.md`, `feature-workflow.prompt.md`, and the `README.md`. All input/output contracts, memory patterns, and dispatch points documented.
- **gaps:** No code or runtime was examined (there is none — system is pure markdown agent definitions). Platform-level capabilities of the GitHub Copilot agent runtime (timer access, tool restriction mechanism) are assumed, not verified.
