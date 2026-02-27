# Research: Impact Analysis

## Focus Area

**impact** — Affected components, modules, and agents; what needs to change and where; what to keep, merge, remove, or introduce.

## Summary

The Forge pipeline has 23 agent files with significant redundancy across CT (4 agents), V (4 agents), and R (4 agents) clusters; Anvil's adversarial multi-model review, evidence gating, and risk classification can replace or subsume 8–10 Forge agents while improving verification rigor; the memory system and artifact evaluation system add substantial overhead that newer patterns (typed schemas, selective gating) would reduce.

---

## Findings

### 1. Forge Agent Inventory and Value Assessment

The Forge system at `.github/agents/` contains 23 files (21 active agent definitions, 1 deprecated agent, 2 reference docs) plus 1 prompt file. The full inventory with assessed value:

#### Core Pipeline Agents (5 — HIGH VALUE, keep with restructuring)

| Agent                   | Lines | Role                                                                                                                                  | Value Assessment                                                                                                                                                                                                              |
| ----------------------- | ----- | ------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `orchestrator.agent.md` | 544   | Central coordinator: 8-step pipeline, memory lifecycle, cluster dispatch (Patterns A/B/C), telemetry tracking, cluster decision logic | Essential but overly complex. Single file coordinates everything. Contains decision logic for 3 clusters (CT, V, R), memory merge scheduling, retry budgets, concurrency caps. The 544-line prompt is pushing context limits. |
| `researcher.agent.md`   | ~150  | Codebase investigation. Dispatched ×4 in parallel with different focus areas                                                          | High value. Post-mortem scores: avg usefulness 9.0 (architecture), 9.0 (impact), 8.0 (dependencies), 10.0 (patterns). Zero inaccuracies across all 4 instances.                                                               |
| `spec.agent.md`         | ~180  | Feature specification from research                                                                                                   | Needed but weakest producer. Post-mortem: avg usefulness 6.6, 12 reported inaccuracies. Primary issue: missing phase/scope annotations caused 6-agent cascade.                                                                |
| `designer.agent.md`     | ~200  | Technical design from spec                                                                                                            | Good value. Post-mortem: avg usefulness 8.5, 1 inaccuracy. Required 3 dispatches due to CT iteration loop.                                                                                                                    |
| `planner.agent.md`      | 275   | Task decomposition with dependency-aware waves, pre-mortem analysis, replan mode                                                      | High value. Post-mortem: avg usefulness 9.0, clarity 9.4, 1 inaccuracy. Dependency-aware wave pattern is strong.                                                                                                              |

#### Implementation Agents (2 — HIGH VALUE, keep)

| Agent                           | Lines | Role                                          | Value Assessment                                                                                                        |
| ------------------------------- | ----- | --------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| `implementer.agent.md`          | 236   | TDD-based task implementation                 | Essential. Well-scoped with clear TDD workflow, security rules, and code quality principles.                            |
| `documentation-writer.agent.md` | ~160  | Documentation generation (read-only codebase) | Low utilization. Only dispatched for documentation-specific tasks. Could be folded into implementer for doc-type tasks. |

#### CT Cluster (4 active + 1 deprecated — RESTRUCTURE/MERGE)

| Agent                         | Lines | Scope                                                             | Value Assessment                                                                                                                                          |
| ----------------------------- | ----- | ----------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `critical-thinker.agent.md`   | ~170  | DEPRECATED — superseded by 4 CT sub-agents                        | Remove entirely. Retained only for migration reference.                                                                                                   |
| `ct-security.agent.md`        | ~210  | Security vulnerabilities + backwards compatibility                | Overlaps with r-security (Step 7) and with Anvil's adversarial security review. Duplicates security analysis at two pipeline stages.                      |
| `ct-scalability.agent.md`     | ~200  | Scalability bottlenecks + performance implications                | Narrowly scoped. In the recorded run, its findings were addressed within 1 CT iteration. Could fold into a broader critical review.                       |
| `ct-maintainability.agent.md` | ~200  | Maintainability, complexity, integration risks                    | Similarly narrow scope. Significant prompt duplication with siblings (~50% shared Mindset + Operating Rules content).                                     |
| `ct-strategy.agent.md`        | ~200  | Strategic risks, scope, edge cases, fundamental approach validity | Broadest CT agent and most valuable. Explicitly asks "Is this the right approach at all?" — the other 3 agents' scopes are subsets of strategic thinking. |

**CT Cluster Findings:**

- The 4 CT agents share ~50% of their prompt content (Mindset section, Operating Rules, output format structure)
- In the only recorded run, the CT cluster required **3 full iterations** (spanning 5 time steps T+3 to T+7), making it the primary pipeline bottleneck
- Each CT iteration dispatches 4 agents in parallel — 12 CT agent invocations total for one design review
- ct-security overlaps with r-security (same domain, different pipeline stage)
- ct-scalability and ct-maintainability probe narrower concerns that ct-strategy already partially covers
- Anvil's adversarial multi-model review (3 different LLMs reviewing the same code) provides genuine diverse perspectives vs. Forge CT's same-model-different-prompt approach

#### V Cluster (4 — PARTIALLY MERGE)

| Agent                | Lines | Scope                                                        | Value Assessment                                                                                                                                                                                              |
| -------------------- | ----- | ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `v-build.agent.md`   | ~180  | Build system detection + compilation gate                    | Essential sequential gate. Binary pass/fail. Post-mortem: avg usefulness 9.0, zero inaccuracies. Keep.                                                                                                        |
| `v-tests.agent.md`   | ~200  | Full test suite execution + cross-task integration detection | Valuable. Tests are the ground truth signal. Keep but could merge with v-tasks.                                                                                                                               |
| `v-tasks.agent.md`   | ~200  | Per-task acceptance criteria verification                    | Overlaps with v-feature (both verify acceptance criteria, at different granularity). The distinction between "per-task" and "feature-level" acceptance criteria is valid but may not justify separate agents. |
| `v-feature.agent.md` | 202   | Feature-level acceptance criteria + regression check         | Overlaps with v-tasks. Feature-level criteria = union of task-level criteria in most cases. Regression check is the unique differentiator.                                                                    |

**V Cluster Findings:**

- v-build as a gate is architecturally sound — keep
- v-tests provides ground-truth test signals — keep
- v-tasks and v-feature both verify acceptance criteria at different granularity levels; the separation creates overhead (2 agents, 2 memory files, 2 evaluation files) without proportional value
- Anvil's approach: single verification cascade (IDE diagnostics → build → type check → lint → tests → smoke) covers all levels in one pass with SQL-tracked evidence
- Anvil's verification is **tool-call evidence, not prose claims**. The INSERT-then-report pattern prevents hallucinated verification. Forge V agents can only inspect code — they don't execute verification themselves (except v-build and v-tests which run commands)
- v-tasks relies on code inspection to verify acceptance criteria — this is weaker than actually running checks

#### R Cluster (4 — REPLACE with Anvil-style adversarial review)

| Agent                  | Lines | Scope                                                        | Value Assessment                                                                                                                                                                                                          |
| ---------------------- | ----- | ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `r-quality.agent.md`   | 229   | Code quality, readability, DRY/KISS/YAGNI                    | Standard code review. Anvil's adversarial reviewers (different models) provide this as part of their general review.                                                                                                      |
| `r-security.agent.md`  | 270   | Security scanning, OWASP compliance, pipeline blocker        | Overlaps with ct-security. The pipeline-blocker override rule (Blocker severity → pipeline ERROR) is the unique value.                                                                                                    |
| `r-testing.agent.md`   | 234   | Test quality and coverage adequacy                           | Standard test review. Anvil's adversarial reviewers also cover this.                                                                                                                                                      |
| `r-knowledge.agent.md` | 340   | Knowledge evolution, instruction/skill updates, decision log | Unique value — no Anvil equivalent. Auto-applies instruction updates, maintains decision log, captures patterns. But it's the longest R agent and modifies files outside the feature directory (`.github/instructions/`). |

**R Cluster Findings:**

- r-quality, r-security, and r-testing all do single-model code review from different angles — this is exactly what Anvil's multi-model adversarial review does, but Anvil uses **genuinely different models** (gpt-5.3-codex, gemini-3-pro-preview, claude-opus-4.6) for actual perspective diversity
- r-security's pipeline-blocker override is valuable but doesn't need to be a separate agent — it's a policy decision (security findings block pipeline) that can be applied to any review output
- r-knowledge is unique and valuable for knowledge evolution, but its auto-apply pattern (modifying `.github/instructions/`) outside the feature scope needs governance audit
- The R cluster produces 4 agents' worth of overhead (memory files, evaluation files, orchestrator decision logic) for what Anvil achieves with 1–3 subagent `code-review` calls

#### Support Agents & Reference Docs

| File                   | Role                                                                        | Value Assessment                                                                                                                                                                                                                    |
| ---------------------- | --------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `post-mortem.agent.md` | 250 lines. Quantitative metrics from evaluations + telemetry. Non-blocking. | Useful for pipeline learning but high overhead relative to value. Requires Global Rule 13 telemetry accumulation, all 14 agents to produce evaluations, evaluation-schema.md reference. Output is consumed by humans, not pipeline. |
| `evaluation-schema.md` | Shared YAML schema for artifact evaluations                                 | Supports post-mortem. Created the "centralized schema reference" pattern. But requires each of 14 agents to include an evaluation step → ~14 extra evaluation operations per pipeline run.                                          |
| `dispatch-patterns.md` | Reference doc for Patterns A/B/C                                            | Useful reference. Content should be absorbed into the next-gen orchestrator's design rather than kept as a separate file.                                                                                                           |

---

### 2. Anvil Components — Impact of Incorporation

#### 2a. Adversarial Multi-Model Review (HIGH IMPACT)

**What it is:** Anvil dispatches `code-review` subagents using different LLM models (gpt-5.3-codex, gemini-3-pro-preview, claude-opus-4.6) to review the same staged changes in parallel. For Medium tasks: 1 reviewer. For Large/🔴 tasks: 3 reviewers.

**Impact on Forge:**

- **Replaces:** CT cluster (4 agents doing same-model-different-prompt review) + R cluster's r-quality, r-security, r-testing (3 agents doing same-model-different-focus review)
- **Net reduction:** 7 agents → 1–3 subagent calls
- **Quality improvement:** Genuine model diversity (different training data, different reasoning patterns, different blind spots) vs. prompt-based specialization within same model
- **Evidence:** Anvil's 5c gate (verify INSERT count before presentation) prevents hallucinated reviews. Forge has no comparable gate on review reliability.
- **Caveat:** Multi-model dispatch requires VS Code subagent model routing support. The `agent_type: "code-review", model: "..."` pattern assumes model selection is controllable — this is platform-dependent.

**Files affected:** All CT agent files (4), r-quality, r-security, r-testing, orchestrator (CT/R cluster decision logic sections), dispatch-patterns.md (Pattern A)

#### 2b. SQL Verification Ledger (HIGH IMPACT)

**What it is:** Anvil creates an `anvil_checks` SQLite table and INSERTs every verification step (baseline, after, review). The evidence bundle is a SELECT query, not prose. "If the INSERT didn't happen, the verification didn't happen."

**Impact on Forge:**

- **Replaces/enhances:** V cluster's prose-based verification. Currently v-build and v-tests run commands and write Markdown reports. The reports are text — no structural guarantee of completeness.
- **Advantage:** SQL-tracked verification is tamper-evident and query-able. Prevents the "I checked and it passed" hallucination pattern.
- **Caveat:** Requires SQLite availability in the runtime environment. Not all VS Code agent environments guarantee database access.
- **Hybrid approach:** Use SQL where available, with a structured YAML fallback (typed schemas per article)

**Files affected:** v-build, v-tests, v-tasks, v-feature (verification output format), implementer (verification cascade adoption)

#### 2c. Risk Classification System (HIGH IMPACT)

**What it is:** Anvil classifies every file change as:

- 🟢 Additive changes, new tests, documentation, config
- 🟡 Modifying existing business logic, function signatures, DB queries
- 🔴 Auth/crypto/payments, data deletion, schema migrations, concurrency, public API

Risk level determines verification depth: 🔴 files escalate to Large task (3 reviewers).

**Impact on Forge:**

- **New capability:** Forge has no per-file risk classification. Verification depth is uniform regardless of change risk.
- **Drives:** Verification depth (number of review models), gate stringency, reviewer allocation
- **Files affected:** Planner (assign risk levels during task decomposition), implementer (adopt risk-aware verification), orchestrator (risk-based verification dispatch)

#### 2d. Evidence Gating (HIGH IMPACT)

**What it is:** Anvil has explicit gates that block progression:

- Gate before Step 4 (implement): baseline INSERTs must be complete
- Gate before 5d (presentation): review verdicts must be INSERTed
- Gate before evidence bundle: ≥2 (Medium) or ≥3 (Large) after-phase verification signals

**Impact on Forge:**

- **Replaces:** Forge's completion contract (DONE/NEEDS_REVISION/ERROR) is the current gating mechanism, but it relies on agent self-report, not evidence verification
- **Enhancement:** Add evidence gates at key pipeline transitions: post-build, post-implementation, post-verification
- **Files affected:** Orchestrator (add evidence verification steps), implementer (adopt gating before reporting DONE)

#### 2e. Baseline Capture (MEDIUM IMPACT)

**What it is:** Before making any code changes, Anvil captures the current system state (IDE diagnostics, build result, test results) as `phase = 'baseline'` entries.

**Impact on Forge:**

- **New capability:** Forge's V-Build runs the build after implementation, but there's no before-and-after comparison. Baseline capture enables regression detection by comparison rather than code inspection.
- **Files affected:** Implementer (add baseline capture before changes), v-build (compare against baseline)

#### 2f. Pushback System (MEDIUM IMPACT)

**What it is:** Anvil evaluates whether a request is a good idea at both implementation AND requirements levels before executing. Surfaces `⚠️ Anvil pushback` when seeing tech debt, simpler approaches, scope issues, or dangerous edge cases.

**Impact on Forge:**

- **New capability:** Forge's pipeline executes the user's request without questioning it. The spec agent could incorporate pushback, or it could be a pre-pipeline gate.
- **Files affected:** New pushback phase or integration into spec agent

#### 2g. Task Sizing and Confidence Scoring (MEDIUM IMPACT)

**What it is:** Anvil sizes tasks (Small/Medium/Large) to determine verification depth. Confidence scoring (High/Medium/Low) with clear pass-on definitions tells the user how much to trust the output.

**Impact on Forge:**

- **Partially exists:** Planner assigns effort (Low/Medium) but this doesn't drive verification depth
- **Enhancement:** Connect task sizing to verification depth and reviewer count
- **Files affected:** Planner (sizing), orchestrator (dispatch logic), implementer (verification cascade)

---

### 3. Memory System Impact

#### Current State

The Forge memory system creates:

- `memory.md` (shared, orchestrator-sole-writer, ~20+ sections)
- `memory/<agent>.mem.md` files (one per agent invocation, ~20+ files per run)
- Memory merge operations after each agent/cluster (~12 merges per run)
- Memory prune operations at 3 checkpoints
- Memory invalidation on revision
- Memory validation after each agent

**Overhead quantification (from the recorded run):**

- 20+ memory files created
- ~12 merge operations (each requiring a subagent dispatch)
- 3 prune operations
- Multiple invalidation/validation cycles
- The memory merge subagent is dispatched repeatedly for what is essentially "copy key findings from A to B"

#### Article Principle: Typed Schemas

The article's first principle states: "Natural language is messy; typed schemas make inter-agent communication reliable." The current `.mem.md` files use semi-structured Markdown with expected sections (Status, Key Findings, Highest Severity, etc.) — this is a weak typing layer over natural language.

**Impact of redesign:**

- Replace `.mem.md` Markdown files with typed schemas (YAML with validated fields)
- Replace prose memory merge with structured contract passing
- Eliminate the need for a separate "merge subagent" — orchestrator can read typed artifacts directly
- Reduce memory file count from ~20 to the minimum needed for cross-agent state

**Files affected:** All 21 active agent definitions (memory output format), orchestrator (memory lifecycle, merge operations, cluster decision logic), memory.md template

---

### 4. Artifact Evaluation System Impact

#### Current State

- 14 agents produce `artifact_evaluation` YAML blocks per the shared schema
- Creates 16 evaluation files with 39 total blocks (per the recorded run)
- PostMortem agent aggregates these into quantitative metrics
- Each agent has an "Evaluate Upstream Artifacts" workflow step adding ~10–15 lines of prompt and one execution step

#### Value vs. Cost

- **Value:** Identified spec as the weakest producer (6.6 usefulness), which is actionable. Identified 14 inaccuracies traceable to feature.md. This is genuine pipeline learning.
- **Cost:** 14 agents × 1 evaluation step = 14 extra operations per run. 16 evaluation files created. PostMortem agent to process them. Plus the schema reference document. Total overhead: ~1 additional agent dispatch + context window budget across 14 agents.

#### Impact of redesign:

- **Option A (keep):** Retain as-is for full pipeline learning
- **Option B (selective):** Evaluate only at key handoff points (research→spec, spec→designer, designer→planner, implementation→V cluster, V cluster→R review) — reduces from 14 to ~5 evaluation points
- **Option C (remove):** Drop entirely. Accept loss of pipeline learning signal. Use the adversarial review's findings instead.
- Recommendation for design phase: Option B (selective evaluation at handoff points) balances learning with cost

**Files affected (if reducing):** 9 agents would lose evaluation steps, evaluation-schema.md scope narrows, post-mortem.agent.md input changes

---

### 5. Component Removal Impact

#### 5a. "Pending Step" Pattern

- **What:** User confirmed this is no longer needed
- **Current location:** Not present in current codebase (may have been removed already, or is referenced in an older version)
- **Impact:** None — already absent or trivially removable

#### 5b. Deprecated `critical-thinker.agent.md`

- **Current state:** File contains deprecation notice at top, references migration to CT cluster
- **Impact of removal:** Delete 1 file. No references from orchestrator or prompt.

#### 5c. Excessive Memory Merge Overhead

- **Impact:** Reducing 12+ merge operations to typed schema passing would simplify orchestrator by ~100 lines (memory lifecycle sections: Initialize, Merge, Prune, Extract Lessons, Invalidate, Clean, Validate, PostMortem merge)
- **Risk:** Agents that depend on `memory.md` for orientation would need alternative context passing

#### 5d. Separate Artifact Evaluations (if reducing to selective)

- **Impact:** Remove evaluation step from ~9 agent definitions. Simplify PostMortem or remove it entirely. Delete evaluation-schema.md or narrow scope.
- **Risk:** Loss of per-agent accuracy tracking. But adversarial review provides a stronger verification signal.

#### 5e. Post-Mortem Agent

- **Impact of removal:** Delete 1 agent file (~250 lines). Remove Step 8 from orchestrator (~40 lines). Remove Global Rule 13 telemetry tracking (~15 lines). Remove 3 doc structure entries. Simplify prompt file.
- **Risk:** Loss of quantitative pipeline metrics. But the data is retrospective — it improves the next run, not the current one.
- **Alternative:** Fold post-mortem metrics into r-knowledge's scope (it already does knowledge evolution)

---

### 6. New Components Impact

#### 6a. Adversarial Phase (replaces CT cluster + partial R cluster)

- **Where in pipeline:** After design (Step 3), replacing Step 3b (CT cluster)
- **What it does:** Dispatch 1–3 adversarial reviewers using different LLM models to review design.md
- **For Large/🔴:** 3 models in parallel (same prompt, different models)
- **For Standard:** 1 or 2 models
- **Impact:** Replaces 4 CT agent files + CT decision logic in orchestrator. May also replace r-quality/r-security/r-testing at Step 7 if applied to code review too.
- **New files needed:** 1 adversarial-review agent definition (parameterized by model and scope — design review or code review)

#### 6b. Evidence Gating Mechanism

- **Where:** Orchestrator pipeline transitions
- **Gates needed:**
  1. Post-research: evidence that codebase was actually investigated (not just claimed)
  2. Post-design: adversarial review verdicts recorded
  3. Post-implementation: build passes + unit tests pass (evidence, not prose)
  4. Post-verification: integration verification evidence
- **Impact:** Adds gate logic to orchestrator at 3–4 pipeline points. Requires structured evidence format.

#### 6c. Risk Classification System

- **Where:** Planner + implementer + verification dispatch
- **What:** Classify every file change as 🟢/🟡/🔴, determining verification depth
- **Impact:** Planner adds risk column to task files. Implementer reads risk level for verification depth. Orchestrator uses risk level for reviewer count dispatching.

#### 6d. Structured Multiple-Choice Interaction

- **Where:** Approval gates (post-research, post-planning)
- **What:** Instead of free-form approval requests, present structured multiple-choice options (from Anvil's `ask_user` pattern)
- **Impact:** Orchestrator approval gates at Steps 1.1a and 4a. Minor enhancement to existing APPROVAL_MODE logic.

---

### 7. Agent Count Impact Summary

| Category                                                          | Current (Forge)         | Proposed Direction                                        | Net Change    |
| ----------------------------------------------------------------- | ----------------------- | --------------------------------------------------------- | ------------- |
| Core pipeline (orchestrator, researcher, spec, designer, planner) | 5                       | 5 (restructured)                                          | 0             |
| Implementation (implementer, doc-writer)                          | 2                       | 1 (merge doc-writer into implementer)                     | -1            |
| CT cluster                                                        | 4 active + 1 deprecated | 0 (replaced by adversarial multi-model review)            | -5            |
| V cluster                                                         | 4                       | 2 (v-build + merged v-verify)                             | -2            |
| R cluster                                                         | 4                       | 1–2 (r-knowledge kept + optional adversarial code review) | -2 to -3      |
| Post-mortem                                                       | 1                       | 0 (fold into r-knowledge or remove)                       | -1            |
| **New: adversarial reviewer**                                     | 0                       | 1 (parameterized)                                         | +1            |
| **New: evidence gating/pushback**                                 | 0                       | 0–1 (logic in orchestrator or separate)                   | 0 to +1       |
| **Total**                                                         | 21 active + 2 reference | ~10–12 agents + 1–2 reference                             | **-9 to -11** |

---

## File References

| File/Folder                                                                                                             | Relevance                                                                                                                      |
| ----------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `.github/agents/orchestrator.agent.md`                                                                                  | Central coordinator — most impacted file. Memory lifecycle, cluster dispatch, decision logic all change.                       |
| `.github/agents/ct-security.agent.md`, `ct-scalability.agent.md`, `ct-maintainability.agent.md`, `ct-strategy.agent.md` | CT cluster — candidates for replacement by adversarial multi-model review                                                      |
| `.github/agents/critical-thinker.agent.md`                                                                              | Deprecated — remove                                                                                                            |
| `.github/agents/r-quality.agent.md`, `r-security.agent.md`, `r-testing.agent.md`                                        | R cluster review agents — candidates for replacement by adversarial code review                                                |
| `.github/agents/r-knowledge.agent.md`                                                                                   | Knowledge evolution — unique value, keep with governance audit                                                                 |
| `.github/agents/v-tasks.agent.md`, `v-feature.agent.md`                                                                 | V cluster — candidates for merging into unified verifier                                                                       |
| `.github/agents/v-build.agent.md`, `v-tests.agent.md`                                                                   | V cluster — keep (essential gates)                                                                                             |
| `.github/agents/post-mortem.agent.md`                                                                                   | PostMortem — candidate for removal or folding into r-knowledge                                                                 |
| `.github/agents/evaluation-schema.md`                                                                                   | Evaluation system — scope narrows if selective evaluation adopted                                                              |
| `.github/agents/documentation-writer.agent.md`                                                                          | Low utilization — candidate for merge into implementer                                                                         |
| `.github/agents/dispatch-patterns.md`                                                                                   | Reference doc — absorb content into redesigned orchestrator                                                                    |
| `.github/agents/implementer.agent.md`                                                                                   | Implementation — enhance with Anvil's verification cascade, risk classification, baseline capture                              |
| `.github/agents/researcher.agent.md`                                                                                    | Research — high value, minimal changes                                                                                         |
| `.github/agents/spec.agent.md`                                                                                          | Spec — enhance with pushback system                                                                                            |
| `.github/agents/designer.agent.md`                                                                                      | Design — minimal changes                                                                                                       |
| `.github/agents/planner.agent.md`                                                                                       | Planning — add risk classification to task decomposition                                                                       |
| `.github/prompts/feature-workflow.prompt.md`                                                                            | Entry point — complete rewrite to match new pipeline                                                                           |
| `Anvil/anvil.agent.md`                                                                                                  | Source system for adversarial review, evidence gating, risk classification, pushback, verification cascade, confidence scoring |
| `docs/feature/orchestrator-tool-restriction/post-mortems/2025-07-17-post-mortem.md`                                     | Real-world performance data informing agent value assessment                                                                   |
| `docs/feature/orchestrator-tool-restriction/decisions.md`                                                               | Prior architectural decisions (tool restriction, memory disambiguation)                                                        |
| `docs/feature/self-improvement-system/decisions.md`                                                                     | Prior decisions (evaluation schema, telemetry, PostMortem scope)                                                               |

---

## Assumptions & Limitations

1. **Single pipeline run data:** All quantitative assessments (agent accuracy scores, bottleneck identification, evaluation counts) are based on a single recorded run (orchestrator-tool-restriction, 2025-07-17). More runs would provide stronger statistical evidence.
2. **Model routing assumption:** Anvil's multi-model adversarial review assumes the runtime can route subagent calls to specific LLM models (gpt-5.3-codex, gemini-3-pro-preview, claude-opus-4.6). This capability is platform-dependent and may not be available in all VS Code agent environments.
3. **SQL availability:** Anvil's verification ledger assumes SQLite access. Not all environments provide this.
4. **Documentation-only codebase:** Both Forge and Anvil are prompt-engineering systems (Markdown agent definitions), not traditional code. "Impact" refers to agent definition files and workflow structure, not compiled code.
5. **Agent count ≠ cost:** Fewer agents doesn't automatically mean lower cost. A single complex agent may consume more context than 4 simple ones. The goal is reducing orchestration overhead while preserving verification rigor.

---

## Open Questions

1. **Multi-model support:** Does the VS Code Copilot runtime actually support dispatching subagents to specific LLM models? If not, the adversarial multi-model review pattern needs an alternative implementation.
2. **SQL in agent runtime:** Can agents reliably create and query SQLite databases in the VS Code extension host? The verification ledger pattern depends on this.
3. **R-Knowledge governance:** R-Knowledge auto-applies instruction file changes (`.github/instructions/`). In the next-gen system, should this remain autonomous or require human approval?
4. **Evaluation retention threshold:** If moving to selective evaluation (Option B), which handoff points provide the highest signal-to-noise ratio? The spec→designer handoff had the most inaccuracy signal in the recorded run.
5. **Memory system replacement:** If replacing `.mem.md` files with typed schemas, what is the transport mechanism? Structured YAML in files? JSON in subagent return values? In-memory context passing?
6. **PostMortem value proposition:** Is retrospective pipeline learning worth the per-run overhead? The one recorded run generated actionable data (spec weakness, CT bottleneck), but humans must act on it manually.

---

## Research Metadata

- **confidence_level:** high — all 23 agent files plus Anvil were read in full; one complete pipeline run with post-mortem data was analyzed
- **coverage_estimate:** Complete coverage of all Forge agent files, Anvil agent file, prompt file, reference docs, prior feature decisions, and one complete pipeline run's post-mortem and telemetry data
- **gaps:** Only one pipeline run available for quantitative analysis. No direct measurement of orchestration token cost per pipeline run (memory merge overhead, evaluation overhead in tokens). No runtime testing of multi-model dispatch feasibility or SQLite availability in the VS Code agent environment.
