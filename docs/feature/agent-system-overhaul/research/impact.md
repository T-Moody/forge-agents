# Research: Impact

## Summary

NewAgents (12 agents) represents a major consolidation from Forge (28 agents), eliminating 16 agent files. The three largest architectural changes are: (1) memory system completely removed in favor of typed YAML routing, (2) evaluation/rating system absent — no quantitative agent improvement data, (3) specialized verification and review clusters collapsed into per-task single agents. The SQL evidence gate pattern and typed YAML schemas are strict improvements over Forge's prose-based approach. Key gaps for the overhaul: no artifact evaluation system, no PostMortem consumer for telemetry DB, no feature-level integration verification, and the no-file-redirect rule is agent-specific rather than global. The instruction file update capability exists in agent definitions but has no instruction files to operate on.

## Findings

### F-1: Agent Inventory Gap — NewAgents Has 12 Files vs Forge's 28

**Category:** structure

NewAgents defines 12 agent/reference files: adversarial-reviewer, designer, dispatch-patterns, implementer, knowledge-agent, orchestrator, planner, researcher, schemas, severity-taxonomy, spec, verifier.

Forge defines 28 files. The **16 files missing** from NewAgents are:

| Category                  | Missing Agents                                                                                            |
| ------------------------- | --------------------------------------------------------------------------------------------------------- |
| Critical Thinkers (5)     | critical-thinker.agent.md (deprecated base), ct-security, ct-scalability, ct-maintainability, ct-strategy |
| Specialized Reviewers (4) | r-quality, r-security, r-testing, r-knowledge                                                             |
| Specialized Verifiers (4) | v-build, v-feature, v-tasks, v-tests                                                                      |
| Other (3)                 | documentation-writer, evaluation-schema, post-mortem                                                      |

In NewAgents:

- CT cluster (4 agents) → 3 adversarial-reviewer instances with multi-model voting
- V cluster (4 verifiers) → single per-task verifier with 4-tier cascade
- R cluster (4 reviewers) → adversarial-reviewer instances at Step 7 + knowledge-agent at Step 8
- Documentation writing → absorbed into implementer (`task_type: documentation` mode)

**Evidence:**

- `NewAgents/.github/agents/` — 12 files total
- `Forge/.github/agents/` — 28 files total
- `NewAgents/.github/agents/implementer.agent.md` lines 276-300 — documentation mode
- `NewAgents/.github/agents/orchestrator.agent.md` lines 283-345 — adversarial-reviewer replaces CT
- `NewAgents/.github/agents/orchestrator.agent.md` lines 420-460 — single verifier replaces V cluster

**Relevance:** Defines the full scope of what was consolidated or dropped. Any overhaul must decide which of these 16 capabilities to restore, merge, or intentionally omit.

---

### F-2: Artifact Evaluation/Rating System Completely Absent

**Category:** risk

Forge has `evaluation-schema.md` defining a structured rating system:

- `usefulness_score` (1-10)
- `clarity_score` (1-10)
- `useful_elements`, `missing_information`, `information_not_used`, `inaccuracies`, `impact_on_work`

All 14 evaluating agents in Forge produce evaluation files per upstream artifact consumed. These feed into PostMortem's quantitative agent-accuracy analysis.

NewAgents has **zero** evaluation infrastructure: no evaluation-schema.md, no agent produces `artifact_evaluation` blocks, no downstream consumer processes ratings.

**Impact of adding:** Would require modifying every consuming agent (spec, designer, planner, implementer, verifier, adversarial-reviewer, knowledge-agent — minimum 7 agents) to produce evaluation blocks. Would also require adding a consumer (PostMortem or Knowledge Agent extension) to process them.

**Evidence:**

- `Forge/.github/agents/evaluation-schema.md` — full schema with scoring fields
- `Forge/.github/agents/documentation-writer.agent.md` lines 73-86 — evaluation workflow example
- `Forge/.github/agents/post-mortem.agent.md` lines 73-90 — PostMortem reads evaluations
- `NewAgents/.github/agents/schemas.md` — no evaluation schema among 10 schemas
- `NewAgents/.github/agents/knowledge-agent.agent.md` — no evaluation consumption

**Relevance:** Without evaluation data, the pipeline cannot quantitatively identify which agents produce low-quality artifacts or which artifact types waste downstream context budget. This is the primary data source for pipeline self-improvement.

---

### F-3: Pipeline Telemetry DB Exists but Has No PostMortem Consumer

**Category:** risk

NewAgents creates `pipeline-telemetry.db` at Step 0 with a `pipeline_telemetry` table (run_id, step, agent, instance, started_at, completed_at, status, dispatch_count, retry_count, notes). The Knowledge Agent at Step 8 lists it as an input but only produces a minimal `pipeline_telemetry_summary` (total_dispatches, total_duration_seconds, error_count).

Forge approach: Orchestrator accumulates telemetry in-context, passes structured table to PostMortem at Step 8. PostMortem outputs:

- `agent-metrics/<run-date>-run-log.md` — per-agent structured YAML telemetry
- `post-mortems/<run-date>-post-mortem.md` — quantitative analysis with `agent_accuracy_scores`, `recurring_issues`, `bottlenecks`, `most_common_missing_information`

NewAgents approach: Telemetry stored in SQLite DB (structural improvement — survives context overflow) but aggregation into detailed run-logs and post-mortem reports is missing.

**Evidence:**

- `NewAgents/.github/agents/orchestrator.agent.md` lines 165-185 — pipeline_telemetry table DDL
- `NewAgents/.github/agents/knowledge-agent.agent.md` line 60 — pipeline-telemetry.db listed as input
- `NewAgents/.github/agents/knowledge-agent.agent.md` lines 70-100 — minimal telemetry_summary
- `Forge/.github/agents/post-mortem.agent.md` lines 1-200 — detailed PostMortem workflow

**Relevance:** The telemetry DB is a structural improvement over Forge's in-context approach, but without a proper consumer, the data is write-only. The overhaul must either extend Knowledge Agent or add a PostMortem step.

---

### F-4: Memory System Completely Removed — Dual-Layer Memory Replaced by Typed YAML

**Category:** structure

**Forge memory system:**

- Shared `memory.md`: Operational memory (Artifact Index, Recent Decisions, Lessons Learned, Recent Updates). Orchestrator is sole writer.
- Isolated memory: Each agent writes `memory/<agent>.mem.md` (Status, Key Findings, Highest Severity, Decisions Made, Artifact Index).
- Memory lifecycle: Initialize → Merge → Prune → Extract Lessons → Invalidate on revision → Clean → Validate
- ~12 merge operations per run

**NewAgents memory system:**

- NO `memory.md` (eliminated entirely)
- NO `*.mem.md` files (eliminated entirely)
- Routing via typed YAML completion contracts (status, summary, severity, findings_count, risk_level, evidence_summary)
- In-context state model only

**What was lost:**

- Lessons Learned persistence across pipeline phases
- Artifact Index navigation for targeted reads
- Cross-agent context summaries (Key Findings)
- Memory-first reading pattern (compact orientation before deep reads)

**What was gained:**

- No merge complexity (~12 merge operations eliminated)
- No memory corruption risk
- Simpler orchestrator (fewer responsibilities)
- Deterministic routing from structured data instead of prose parsing

**Evidence:**

- `Forge/.github/agents/orchestrator.agent.md` — Memory-First Protocol (Rule 6), Memory Write Safety (Rule 12), Memory Lifecycle Actions table
- `NewAgents/.github/agents/orchestrator.agent.md` lines 80-110 — pipeline_state in-context only
- `NewAgents/.github/agents/orchestrator.agent.md` Anti-Drift Anchor — no memory management
- `NewAgents/.github/agents/schemas.md` lines 31-38 — routing via typed YAML

**Relevance:** The memory removal was the single largest architectural change. It simplified the orchestrator dramatically but lost the self-improvement feedback loop. Any overhaul re-introducing memory needs to evaluate whether the complexity cost is justified.

---

### F-5: YAML+MD Dual Output — Downstream Agents Consume YAML Only

**Category:** pattern

NewAgents produces paired outputs for 4 artifact types:

| Artifact | YAML (machine)          | MD (human)            |
| -------- | ----------------------- | --------------------- |
| Research | `research/<focus>.yaml` | `research/<focus>.md` |
| Spec     | `spec-output.yaml`      | `feature.md`          |
| Design   | `design-output.yaml`    | `design.md`           |
| Plan     | `plan-output.yaml`      | `plan.md`             |

**Consumption analysis:**

- Orchestrator reads **ONLY typed YAML** for routing (schemas.md: "The orchestrator reads ONLY the typed YAML for routing decisions. The Markdown companions are for human consumption and auditability.")
- Spec reads `research/<focus>.yaml` (spec.agent.md line 6)
- Designer reads `research/*.yaml` and `spec-output.yaml` (designer.agent.md lines 31-34)
- Implementer reads `tasks/<task-id>.yaml` + `design-output.yaml` + `spec-output.yaml`
- **No downstream agent reads the .md companions for routing or work**

In Forge, the opposite: ONLY .md files exist (no typed YAML). Memory .mem.md files serve as lightweight routing.

The duplication is intentional but doubles agent output work. YAML is authoritative; MD is derivative.

**Evidence:**

- `NewAgents/.github/agents/schemas.md` line 38 — "orchestrator reads ONLY the typed YAML"
- `NewAgents/.github/agents/spec.agent.md` line 6 — inputs are `research/<focus>.yaml`
- `NewAgents/.github/agents/designer.agent.md` lines 31-34 — reads `research/*.yaml`
- `Forge/.github/agents/researcher.agent.md` Outputs — only `research/<focus>.md`

**Relevance:** If output effort is a concern, MD companions could be eliminated for machine-consumed artifacts and retained only where humans need them. Alternatively, MD generation could be deferred.

---

### F-6: Instruction File Update Capability Exists but No Instruction Files Exist

**Category:** risk

NewAgents Knowledge Agent has a Governed Updates section for modifying `.github/instructions/`:

- **Interactive mode:** Changes require explicit user approval
- **Autonomous mode:** Changes are logged but NOT auto-applied

Forge r-knowledge auto-applies instruction and skill updates to `.github/instructions/` and `.github/skills/` (with safety filter).

However, **no** `.github/copilot-instructions.md`, `.github/instructions/`, or `.github/skills/` directories exist anywhere in the workspace.

**Evidence:**

- `NewAgents/.github/agents/knowledge-agent.agent.md` lines 371-390 — Governed Updates
- `Forge/.github/agents/r-knowledge.agent.md` lines 10, 143 — auto-apply
- `file_search` for `**/copilot-instructions.md` — 0 results

**Relevance:** For agents to proactively update instructions, the overhaul must create initial instruction file structure. The governance model must be explicitly chosen.

---

### F-7: Terminal Output File-Redirect Prohibition Only in Implementer — Not System-Wide

**Category:** risk

NewAgents implementer has the rule (Operating Rule 6, line 344): "No file-redirect of command output: Never redirect terminal command output to a file (e.g., `command > output.txt`)."

Forge has this in 3 agents: implementer, v-build, v-tests.

In NewAgents, the **verifier**, **adversarial-reviewer**, and **knowledge-agent** — all of which have `run_in_terminal` access — **lack this prohibition**.

**Agents with `run_in_terminal` in NewAgents:**
| Agent | Has no-redirect rule? |
|-------|----------------------|
| Orchestrator | N/A (uses terminal only for SQL/git) |
| Implementer | ✅ Yes (Rule 6) |
| Verifier | ❌ No |
| Adversarial Reviewer | ❌ No |

**Evidence:**

- `NewAgents/.github/agents/implementer.agent.md` line 344 — Rule 6
- `Forge/.github/agents/v-build.agent.md` line 38, `v-tests.agent.md` line 42 — rule present
- `NewAgents/.github/agents/verifier.agent.md` — no file-redirect prohibition found

**Relevance:** Should be made system-wide in the overhaul via共shared reference document, not repeated per-agent.

---

### F-8: Design Review Architecture — CT Cluster (4 Agents) → Adversarial Reviewer (3 Instances)

**Category:** structure

**Forge Step 3b:** 4 CT sub-agents (ct-security, ct-scalability, ct-maintainability, ct-strategy). Coverage by domain specialization. Prose-based severity evaluation from memory files.

**NewAgents Step 3b:** 3 adversarial-reviewer instances with distinct review_focus (security, architecture, correctness). Intended for different LLM models. SQL-backed evidence gate with formal verdict voting (approve/needs_revision/blocker). Security Blocker Policy: any verdict='blocker' → immediate pipeline ERROR.

| Aspect   | Forge CT Cluster                                       | NewAgents Adversarial Review                    |
| -------- | ------------------------------------------------------ | ----------------------------------------------- |
| Agents   | 4 specialized                                          | 3 multi-model instances                         |
| Coverage | Domain (security/scalability/maintainability/strategy) | Perspective (security/architecture/correctness) |
| Routing  | Prose severity from memory files                       | SQL evidence gate with verdict voting           |
| Security | Through ct-security findings                           | Formal blocker policy via SQL                   |

**Evidence:**

- `Forge/.github/agents/orchestrator.agent.md` — CT Cluster Decision Flow
- `NewAgents/.github/agents/orchestrator.agent.md` lines 283-345 — 3 adversarial-reviewer instances
- `NewAgents/.github/agents/orchestrator.agent.md` lines 740-750 — Security Blocker Policy

**Relevance:** The SQL evidence gate from NewAgents is strictly better than prose parsing. The overhaul must decide between domain coverage (Forge) vs perspective diversity (NewAgents) or both.

---

### F-9: Single Verifier with 4-Tier Cascade Replaces V-Cluster of 4 Specialized Verifiers

**Category:** structure

**Forge V-cluster:**

- v-build (gate): compilation verification
- v-tests: test suite, coverage, TDD compliance
- v-tasks: all tasks complete, acceptance criteria met
- v-feature: feature-level end-to-end requirements satisfaction

**NewAgents verifier (per-task):**

- Tier 1 (always): IDE diagnostics, syntax check
- Tier 2 (if tooling): build, type-check, lint, tests
- Tier 3 (if no runtime): import-check, smoke-execution
- Tier 4 (Large only): readiness (observability, degradation, secrets)
- Plus: baseline cross-check via git tags, regression detection, SQL evidence per check

| What's Lost                                        | What's Gained                                     |
| -------------------------------------------------- | ------------------------------------------------- |
| Feature-level integration verification (v-feature) | Per-check SQL evidence recording                  |
| Plan completeness verification (v-tasks)           | Baseline cross-checking via git tags              |
| Cross-task integration testing                     | Tiered verification with dynamic tool detection   |
|                                                    | Minimum signal requirements (2 Standard, 3 Large) |

**Evidence:**

- `Forge/.github/agents/orchestrator.agent.md` — V Cluster with 4 verifiers
- `NewAgents/.github/agents/verifier.agent.md` — per-task 4-tier cascade
- `NewAgents/.github/agents/orchestrator.agent.md` lines 420-460 — per-task dispatch with SQL gate

**Relevance:** Per-task verification alone may miss feature-level integration issues. The overhaul should consider adding a v-feature equivalent for end-to-end verification.

---

### F-10: Test Execution Correctly Uses run_in_terminal — No VS Code Test Tool Misuse

**Category:** pattern

Investigation of test execution patterns:

- **Implementer:** `run_in_terminal` for test execution, `get_errors` for IDE diagnostics
- **Verifier:** `run_in_terminal` for build/test/lint (Tier 2-3), `get_errors`/`ide-get_diagnostics` for IDE (Tier 1)
- **Orchestrator:** Does NOT run tests. `get_errors` explicitly prohibited.

No agent is configured to use VS Code's built-in test runner tools. All test execution goes through `run_in_terminal` with explicit commands.

**Evidence:**

- `NewAgents/.github/agents/implementer.agent.md` lines 134-166 — run_in_terminal for tests
- `NewAgents/.github/agents/verifier.agent.md` lines 230-280 — Tier 2 tests via run_in_terminal
- `NewAgents/.github/agents/orchestrator.agent.md` line 53 — get_errors restricted

**Relevance:** If users report model drift to VS Code test tools, the issue is behavioral, not definitional. Anti-drift anchors should reinforce terminal-based execution.

---

### F-11: Documentation-Writer Consolidated into Implementer

**Category:** structure

Forge has a dedicated `documentation-writer.agent.md` handling:

- API docs (OpenAPI/Swagger)
- Architectural diagrams (Mermaid)
- README updates
- Code-documentation parity verification
- Documentation coverage matrix

NewAgents absorbed this into the implementer via `task_type: documentation` mode. The implementer skips TDD, captures baseline, writes docs, and produces a standard implementation report.

Forge orchestrator's Rule 9 dispatches `implementer` or `documentation-writer` based on task's `agent` field. NewAgents orchestrator has no documentation-writer dispatch path.

**Evidence:**

- `Forge/.github/agents/documentation-writer.agent.md` lines 1-100 — specialized capabilities
- `Forge/.github/agents/orchestrator.agent.md` Rule 9 — valid agents: implementer, documentation-writer
- `NewAgents/.github/agents/implementer.agent.md` lines 276-300 — documentation mode

**Relevance:** Documentation quality may suffer without a dedicated agent. If the overhaul prioritizes lean agent count, the implementer's documentation mode is sufficient; otherwise, restoring the specialized agent should be considered.

---

## Source Files Examined

- `NewAgents/.github/agents/orchestrator.agent.md`
- `NewAgents/.github/agents/implementer.agent.md`
- `NewAgents/.github/agents/verifier.agent.md`
- `NewAgents/.github/agents/researcher.agent.md`
- `NewAgents/.github/agents/schemas.md`
- `NewAgents/.github/agents/designer.agent.md`
- `NewAgents/.github/agents/spec.agent.md`
- `NewAgents/.github/agents/planner.agent.md`
- `NewAgents/.github/agents/knowledge-agent.agent.md`
- `NewAgents/.github/agents/adversarial-reviewer.agent.md`
- `NewAgents/.github/agents/dispatch-patterns.md`
- `NewAgents/.github/agents/severity-taxonomy.md`
- `Forge/.github/agents/orchestrator.agent.md`
- `Forge/.github/agents/evaluation-schema.md`
- `Forge/.github/agents/post-mortem.agent.md`
- `Forge/.github/agents/verifier.agent.md`
- `Forge/.github/agents/researcher.agent.md`
- `Forge/.github/agents/documentation-writer.agent.md`
- `Forge/.github/agents/critical-thinker.agent.md`
- `Forge/.github/agents/r-knowledge.agent.md`
- `Forge/.github/agents/schemas.md`
- `Forge/.github/agents/v-build.agent.md`
- `Forge/.github/agents/v-tests.agent.md`
- `.github/agents/orchestrator.agent.md`

## Research Metadata

- **Confidence Level:** high — all 28 Forge files and 12 NewAgents files were inventoried; key files read in detail
- **Coverage Estimate:** comprehensive coverage of all 8 impact questions from the dispatch; every agent with run_in_terminal access checked for test patterns and file-redirect rules
- **Gaps:** Did not read all 28 Forge agent files in full detail (focused on key agents with direct impact comparisons). Did not examine the Anvil system deeply for this focus area. Did not examine existing feature outputs under `docs/feature/next-gen-multi-agent-system/` to see real-world YAML+MD usage patterns.
