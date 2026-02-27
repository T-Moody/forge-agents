# Research: Patterns

## Focus Area

Patterns — testing strategies, code conventions, error handling, operational concerns, and reusable patterns across both Forge and Anvil systems.

## Summary

Both systems employ sophisticated but fundamentally different verification and error handling patterns; Forge uses a 23-agent pipeline with typed completion contracts, cluster dispatch, and memory-mediated communication, while Anvil uses a single-agent evidence-first loop with SQL-tracked verification, adversarial multi-model review, and risk-based escalation. Both share the principle that verification must be tool-call evidence (not prose assertions), but diverge on granularity, communication medium, and overhead.

---

## Findings

### 1. Forge Patterns

#### 1.1 Completion Contracts

Every Forge agent terminates with exactly one of three states:

| State             | Meaning                                       | Routing                                 |
| ----------------- | --------------------------------------------- | --------------------------------------- |
| `DONE:`           | Agent completed successfully                  | Orchestrator proceeds to next step      |
| `NEEDS_REVISION:` | Work completed but requires upstream revision | Routed per NEEDS_REVISION routing table |
| `ERROR:`          | Unrecoverable failure                         | Retry once, then halt/escalate          |

**Usage by agent category:**

- **Sequential agents** (spec, designer, planner): Return `DONE` or `ERROR` only — they do not produce `NEEDS_REVISION`. (Source: [orchestrator.agent.md](../../.github/agents/orchestrator.agent.md), NEEDS_REVISION Routing Table)
- **CT cluster sub-agents**: Return `DONE` or `ERROR` only — the orchestrator reads their isolated memory severity ratings and determines if the cluster result is `NEEDS_REVISION`. (Source: [ct-security.agent.md](../../.github/agents/ct-security.agent.md))
- **V cluster sub-agents**: V-Build returns `DONE`/`ERROR` (binary gate). V-Tests, V-Tasks, V-Feature can return all three states. The orchestrator applies the V Decision Table. (Source: [v-build.agent.md](../../.github/agents/v-build.agent.md))
- **R cluster sub-agents**: R-Quality, R-Security, R-Testing can return all three states. R-Knowledge returns `DONE` or `ERROR` only (non-blocking). (Source: [r-knowledge.agent.md](../../.github/agents/r-knowledge.agent.md))
- **PostMortem**: Returns `DONE` or `ERROR` only (non-blocking). (Source: [post-mortem.agent.md](../../.github/agents/post-mortem.agent.md))

**Key observation**: Most agents only use `DONE`/`ERROR`. The `NEEDS_REVISION` state is primarily used by cluster sub-agents whose outputs are aggregated by the orchestrator. The orchestrator itself never returns `NEEDS_REVISION` — it handles revision routing internally.

#### 1.2 Cluster Dispatch Patterns

Three reusable dispatch patterns defined in [dispatch-patterns.md](../../.github/agents/dispatch-patterns.md):

**Pattern A — Fully Parallel:**

- Dispatch ≤4 sub-agents concurrently; wait for all; retry errors once.
- Requires ≥2 outputs for decision-making.
- Used by: Research (Step 1.1), CT cluster (Step 3b), R cluster (Step 7).

**Pattern B — Sequential Gate + Parallel:**

- Gate agent runs first (V-Build); on DONE, dispatch remaining ≤3 in parallel.
- On gate ERROR: skip parallel, cluster ERROR.
- Used by: V cluster (Step 6).

**Pattern C — Replan Loop:**

- Wraps Pattern B with iteration: run V cluster → if NEEDS_REVISION → replan → re-implement → re-verify.
- Maximum 3 iterations.
- Used by: Verification-Replan cycle (Steps 5–6).

**Concurrency cap**: Maximum 4 concurrent subagent invocations. Waves with >4 tasks are partitioned into sub-waves of ≤4. (Source: [orchestrator.agent.md](../../.github/agents/orchestrator.agent.md), Global Rule 8)

#### 1.3 Memory Patterns

**Isolated → Shared merge lifecycle:**

1. Each agent writes to `memory/<agent-name>.mem.md` (isolated).
2. Orchestrator reads isolated memories for routing decisions.
3. Orchestrator dispatches a subagent to merge Key Findings, Decisions, and Artifact Index into shared `memory.md`.
4. Downstream agents read `memory.md` first (memory-first reading), then consult Artifact Index for targeted section reads of full artifacts.

**Memory lifecycle actions** (from [orchestrator.agent.md](../../.github/agents/orchestrator.agent.md), Memory Lifecycle Actions table):

- **Initialize**: Step 0, create empty template.
- **Merge**: After each agent/cluster completes.
- **Prune**: After Steps 1.1m, 2m, 4m — remove Recent Decisions/Updates older than 2 phases; preserve Artifact Index and Lessons Learned always.
- **Extract Lessons**: Between implementation waves.
- **Invalidate on revision**: Mark affected entries with `[INVALIDATED — <reason>]`.
- **Clean invalidated**: After revision agent completes.
- **Validate**: After each agent/cluster — check memory file exists (non-blocking).

**Memory write safety**: Orchestrator is sole writer to `memory.md`. All sub-agents write only to their isolated memory files. No concurrent writes possible.

**Memory-first reading pattern**: Every agent reads `memory.md` first, then upstream isolated memories, then uses Artifact Index for targeted reads of full artifacts. This is enforced via Operating Rule 6 in every agent.

**Memory content format** (standardized across all agents):

```markdown
# Memory: <agent-name>

## Status

## Key Findings (≤5 bullets)

## Highest Severity

## Decisions Made

## Artifact Index
```

#### 1.4 Error Handling Patterns

**Retry budget**: 3 internal attempts (1 original + 2 retries) × 2 orchestrator attempts = 6 maximum tool calls per agent. This is stated identically in every agent's Operating Rules. (Source: all agent files, Operating Rule 2)

**Transient vs deterministic failure distinction**:

- Transient errors (network timeout, tool unavailable, rate limit): Retry up to 2 times.
- **Deterministic failures MUST NOT be retried** — same input produces same failure.
- Persistent errors (file not found, permission denied): Include in output and continue.
- Security issues: Flag immediately with `severity: critical`.
- Missing context: Note gap and proceed.

**Cluster-level error handling**:

- CT cluster: <2 available memories → cluster ERROR. (Source: [orchestrator.agent.md](../../.github/agents/orchestrator.agent.md), CT Cluster Decision Flow)
- V cluster: V-Build ERROR → skip parallel, cluster ERROR. Other sub-agent errors handled by V Decision Table. (Source: orchestrator V Decision Flow)
- R cluster: R-Security missing/ERROR → pipeline ERROR. R-Knowledge ERROR → non-blocking. <2 non-knowledge memories → ERROR. (Source: orchestrator R Cluster Decision Flow)

**Non-blocking agents**: PostMortem and R-Knowledge — their ERROR does not block the pipeline.

#### 1.5 NEEDS_REVISION Routing

The orchestrator maintains a routing table ([orchestrator.agent.md](../../.github/agents/orchestrator.agent.md), NEEDS_REVISION Routing Table):

| Source           | Routes To                                       | Max Loops | Escalation                                                     |
| ---------------- | ----------------------------------------------- | --------- | -------------------------------------------------------------- |
| CT cluster eval  | Designer → full CT re-run                       | 1         | Proceed with warning; forward findings as planning constraints |
| V cluster eval   | Planner (REPLAN) → Implementers → full V re-run | 3         | Proceed with findings documented                               |
| R cluster eval   | Implementer(s) for affected tasks               | 1         | Escalate to planner for full replan                            |
| R-Security ERROR | Retry once → Planner if persistent              | 1         | Halt pipeline                                                  |

**Key characteristic**: Revision routing is **asymmetric** — different sources route to different agents with different loop budgets and escalation paths.

#### 1.6 Evaluation Schema

All 14 evaluating agents use a centralized evaluation schema ([evaluation-schema.md](../../.github/agents/evaluation-schema.md)):

- YAML-formatted blocks with `usefulness_score` (1-10), `clarity_score` (1-10), and five list fields (useful_elements, missing_information, information_not_used, inaccuracies, impact_on_work).
- Evaluation is **secondary and non-blocking** — failure must never cause agent ERROR.
- Written to `artifact-evaluations/<agent-name>.md`.
- Collision avoidance: append sequence suffix for re-runs.
- **Error fallback**: `evaluation_error` block written instead of crashing.

**Observed impact** (from post-mortem data): In the orchestrator-tool-restriction run, 16 evaluation files with 39 evaluation blocks were processed. Researcher cluster scored highest (avg 9.0 usefulness, 8.75 clarity, 0 inaccuracies). Spec scored lowest (avg 6.6 usefulness, 12 inaccuracies). This data drove identification of the missing Phase 1/Phase 2 scoping issue.

#### 1.7 Self-Verification Pattern

Multiple agents include explicit self-verification steps before returning:

- **Designer** (Step 12): Verifies design addresses all requirements, acceptance criteria have implementation paths, security considerations addressed, failure modes identified. Fixes gaps before returning.
- **Spec** (Step 6): Verifies acceptance criteria are testable, all FRs have corresponding ACs, edge cases cover failure modes, no contradictory requirements.
- **CT sub-agents** (Step 10): Re-reads review, confirms each finding is grounded in specific technical details, strengthens vague findings.
- **Implementer** (Step 7): Self-reflection for Medium/High effort tasks — checks all requirements addressed, tests cover ACs, code follows conventions.
- **Orchestrator** (Cluster Decision Flows): Self-verification logging after each cluster evaluation — logs severity values and result to memory.md.

**Pattern**: Each agent verifies its own output against its input requirements before returning. The orchestrator also verifies cluster-level decisions via logging.

#### 1.8 Tool Restriction Patterns

The orchestrator is restricted to: `[agent, agent/runSubagent, memory, read_file, list_dir]`. It **MUST NOT** use `create_file`, `replace_string_in_file`, `multi_replace_string_in_file`, `run_in_terminal`, `grep_search`, `semantic_search`, `file_search`, or `get_errors`. This is enforced via Operating Rule 5 and the Anti-Drift Anchor.

Other agents have different tool access profiles:

- **Read-only agents** (all V agents, all R agents, CT agents, researcher): Cannot modify source code. Enforced via Read-Only Enforcement sections.
- **Write agents** (implementer, documentation-writer): Can modify code/docs within task scope only. File boundaries enforced via Operating Rule 4.
- **R-Knowledge**: Can modify `.github/instructions/` and `.github/skills/` files in addition to its review outputs. Safety constraint filter (KE-SAFE-6) prevents weakening safety checks.

#### 1.9 Telemetry Pattern

Execution telemetry is **context-only** during Steps 1–7:

- After each `runSubagent` return, the orchestrator notes dispatch metadata in its working context: agent name, step, dispatch pattern, status, retry count, failure reason, iteration number, human intervention, timestamp.
- **No file writes** for telemetry during active pipeline execution.
- At Step 8, the full telemetry dataset is included in the PostMortem dispatch prompt.
- PostMortem writes the structured YAML run log to `agent-metrics/<run-date>-run-log.md`.

**Rationale**: Avoids file I/O overhead during active pipeline; telemetry only materialized when pipeline is complete.

#### 1.10 Anti-Drift Anchor Pattern

Every Forge agent includes a final "Anti-Drift Anchor" section — a bolded reminder of the agent's identity, responsibilities, and constraints. Format: "**REMEMBER:** You are the **<Agent>**. You <do X>. You never <do Y>. Stay as <agent>."

This is a prompt engineering pattern to prevent model context window drift in long conversations.

---

### 2. Anvil Patterns

#### 2.1 Task Sizing

Three-tier classification drives verification depth ([anvil.agent.md](../../Anvil/anvil.agent.md)):

| Size   | Criteria                                                                    | Verification Depth                                                                 |
| ------ | --------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| Small  | Typo, rename, config tweak, one-liner                                       | Quick Verify (5a + 5b only) — no ledger, no adversarial review, no evidence bundle |
| Medium | Bug fix, feature addition, refactor                                         | Full Anvil Loop with 1 adversarial reviewer                                        |
| Large  | New feature, multi-file architecture, auth/crypto/payments, OR any 🔴 files | Full Anvil Loop with 3 adversarial reviewers + `ask_user` at Plan step             |

**Escalation rule**: 🔴 files automatically escalate to Large regardless of task complexity.

#### 2.2 Risk Classification

Per-file risk assessment using traffic light system:

- 🟢 Additive changes, new tests, documentation, config, comments
- 🟡 Modifying existing business logic, function signatures, DB queries, UI state management
- 🔴 Auth/crypto/payments, data deletion, schema migrations, concurrency, public API surface

Risk classification is **per-file**, not per-task. A task touching both 🟢 and 🔴 files escalates the entire task to Large.

#### 2.3 Verification Cascade (Tiered)

Defense-in-depth verification with three tiers (always run every applicable tier, don't stop at first):

**Tier 1 — Always run:**

1. IDE diagnostics (via `ide-get_diagnostics`)
2. Syntax/parse check

**Tier 2 — Run if tooling exists (discover dynamically):** 3. Build/compile 4. Type checker 5. Linter 6. Tests (full suite or relevant subset)

**Tier 3 — Required when Tiers 1-2 produce no runtime verification:** 7. Import/load test 8. Smoke execution (3-5 line throwaway script)

If Tier 3 infeasible, INSERT `check_name = 'tier3-infeasible'` with explanation (acceptable, but silently skipping is not).

**Minimum signals**: 2 for Medium, 3 for Large. Zero verification is never acceptable.
**Max fix attempts**: 2 per check. If unfixable after 2 attempts, revert changes and INSERT failure.

#### 2.4 SQL Verification Ledger

All verification for Medium and Large tasks is tracked in SQL:

```sql
CREATE TABLE IF NOT EXISTS anvil_checks (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    task_id TEXT NOT NULL,
    phase TEXT NOT NULL CHECK(phase IN ('baseline', 'after', 'review')),
    check_name TEXT NOT NULL,
    tool TEXT NOT NULL,
    command TEXT,
    exit_code INTEGER,
    output_snippet TEXT,
    passed INTEGER NOT NULL CHECK(passed IN (0, 1)),
    ts DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

**Core rule**: "Every verification step must be an INSERT. The Evidence Bundle is a SELECT, not prose. If the INSERT didn't happen, the verification didn't happen."

**Phases**: `baseline` (before changes), `after` (after changes), `review` (adversarial review verdicts).

#### 2.5 Adversarial Multi-Model Review

**Medium (no 🔴 files)**: One `code-review` subagent using `gpt-5.3-codex`.

**Large OR 🔴 files**: Three reviewers in parallel (same prompt, different models):

- `gpt-5.3-codex`
- `gemini-3-pro-preview`
- `claude-opus-4.6`

Each verdict is INSERTed with `phase = 'review'` and `check_name = 'review-{model_name}'`.

**Gate**: Before presenting results, verify `SELECT COUNT(*) FROM anvil_checks WHERE task_id = '{task_id}' AND phase = 'review'` returns ≥1 for Medium or ≥3 for Large. If insufficient, go back.

**Fix and re-review**: If real issues found, fix, re-run 5b AND 5c. Max 2 adversarial rounds. After the second round, INSERT remaining findings as known issues with Confidence: Low.

#### 2.6 Evidence Gating

Three explicit SQL gates in the Anvil Loop:

1. **Baseline gate** (before Step 4): `SELECT COUNT(*) FROM anvil_checks WHERE task_id = '{task_id}' AND phase = 'baseline'` must be >0. "If you have zero rows… you skipped this step. Go back."
2. **Review gate** (before Step 5d): `SELECT COUNT(*) FROM anvil_checks WHERE task_id = '{task_id}' AND phase = 'review'` must be ≥1 for Medium or ≥3 for Large.
3. **Evidence bundle gate** (before presentation): `SELECT COUNT(*) FROM anvil_checks WHERE task_id = '{task_id}' AND phase = 'after'` must be ≥2 for Medium or ≥3 for Large. Review-phase rows don't count.

**Pattern**: Gates are enforced by SQL COUNT queries, not by prose assertions. This makes verification machine-checkable.

#### 2.7 Pushback Pattern

Anvil evaluates request quality before execution at both implementation and requirements levels:

**Implementation concerns**: Tech debt, duplication, complexity, simpler alternatives.
**Requirements concerns**: Conflicting behavior, symptom vs root cause, dangerous edge cases, implicit assumptions.

**Implementation**: Shows `⚠️ Anvil pushback` callout, then calls `ask_user` with structured choices ("Proceed as requested" / "Do it your way instead" / "Let me rethink this"). Does NOT implement until user responds.

#### 2.8 Baseline Capture

Before/after comparison pattern:

1. Before changing any code, run applicable verification checks and INSERT with `phase = 'baseline'`.
2. Minimum baseline: IDE diagnostics on files to change, build exit code, test results.
3. If baseline already broken: note but proceed — not responsible for pre-existing failures, but responsible for not making them worse.
4. Regressions detected by comparing `phase = 'baseline'` to `phase = 'after'`.

#### 2.9 Confidence Scoring

Three levels with concrete definitions:

- **High**: All tiers passed, no regressions, reviewers found zero issues (or only fixed issues). "You'd merge this without reading the diff."
- **Medium**: Most checks passed but: no test coverage for changed path, reviewer concern addressed but uncertain, blast radius not fully verified. "A human should skim the diff."
- **Low**: A check failed that couldn't be fixed, unverifiable assumptions, or reviewer issue that can't be disproved. **If Low, MUST state what would raise it.**

#### 2.10 Git Hygiene

Pre-task git management:

1. **Dirty state check**: `git status --porcelain` — pushback if uncommitted changes from previous task.
2. **Branch check**: Pushback if on `main`/`master` for Medium/Large tasks — recommend branch creation (`anvil/{task_id}`).
3. **Worktree detection**: Note silently if in worktree.
4. **Post-task commit**: Automatic commit with Co-authored-by trailer after presenting for Medium/Large. Small tasks offer choice. Pre-commit SHA captured for rollback.

#### 2.11 Session Recall Pattern

Before planning (Medium/Large), Anvil queries session history for relevant context:

- SQL query for past sessions that touched the same files.
- Subquery for past problems (regression, broke, failed, reverted).
- If past failures found: mention in plan. If past patterns found: follow them.

#### 2.12 Boost Pattern (Prompt Rewriting)

Rewrites the user's prompt into precise specification. Fixes typos, infers target files, expands shorthand, adds implied constraints. Only shows boosted prompt if intent materially changed.

#### 2.13 Build/Test Command Discovery

Dynamic discovery in priority order:

1. Project instruction files (`.github/copilot-instructions.md`, `AGENTS.md`)
2. Previously stored facts from past sessions
3. Detect ecosystem from config files
4. Infer from ecosystem conventions
5. `ask_user` only after all above fail

Once confirmed working, saved with `store_memory`.

#### 2.14 Learn Step

After verification, before presenting:

1. Working build/test command → `store_memory` immediately
2. Codebase pattern found → `store_memory`
3. Reviewer caught missed verification → `store_memory` the gap
4. Fixed a regression introduced → `store_memory` the file + what went wrong

Does NOT store obvious facts, things in project instructions, or facts about just-written code.

---

### 3. Failure Prevention Patterns (from Article Principles)

The initial request documents eight principles from the GitHub blog post. These map to both systems as follows:

#### 3.1 Typed Schemas

**Forge implementation**: Isolated memory file format is standardized but Markdown-based (Status, Key Findings ≤5, Highest Severity, Decisions Made, Artifact Index). Evaluation schema is YAML-typed with integer scores and list constraints. Completion contracts are typed (DONE/NEEDS_REVISION/ERROR).

**Anvil implementation**: SQL verification ledger enforces types via CHECK constraints (`phase IN ('baseline', 'after', 'review')`, `passed IN (0, 1)`). No typed inter-agent communication (single agent).

**Gap in both**: Memory files and artifact communication are Markdown prose, not machine-parseable schemas. Severity taxonomies differ across clusters (Critical/High/Medium/Low for CT, Blocker/Major/Minor for R, PASS/FAIL for V-Build).

#### 3.2 Action Schemas (Constrained Outputs)

**Forge implementation**: Completion contract constrains every agent to exactly one of three return values. Agent output files are structurally defined (e.g., design.md Contents, feature.md Contents). Planner task files have mandatory fields (Task Goal, depends_on, agent, Acceptance Criteria, etc.).

**Anvil implementation**: Evidence Bundle has a fixed structure (Baseline, Verification, Regressions, Adversarial Review, Confidence). SQL schema constrains verification records to defined phases and boolean passed status.

#### 3.3 MCP Enforcement

**Forge implementation**: Not explicitly using MCP for inter-agent contracts. Enforcement is via prose instructions in agent definitions. The orchestrator validates cluster outputs by reading memory files and applying decision flows.

**Anvil implementation**: Uses MCP tools (Context7 for documentation lookup). SQL gates serve as enforcement layers.

#### 3.4 Design for Failure First

**Forge implementation**: Every cluster decision flow includes error paths. Non-blocking agents (PostMortem, R-Knowledge) prevent cascading failures. Pattern C includes max-iteration bounds. Memory is declared non-essential ("memory failure is non-blocking").

**Anvil implementation**: Baseline capture assumes pre-existing failures. Max 2 fix attempts per verification check with automatic revert. Max 2 adversarial rounds before accepting Low confidence. Tier 3 fallback for environments without runtime verification.

---

### 4. Cross-System Pattern Comparison

#### 4.1 Verification Philosophy

| Aspect                | Forge                                        | Anvil                                              |
| --------------------- | -------------------------------------------- | -------------------------------------------------- |
| Verification unit     | Feature-level (V cluster)                    | Task-level (per change)                            |
| Evidence format       | Markdown prose reports                       | SQL records + tool-call evidence                   |
| Machine verifiability | Low (human-readable)                         | High (SQL-queryable)                               |
| Baseline comparison   | No (V agents check post-implementation only) | Yes (baseline capture before changes)              |
| Adversarial review    | No (CT reviews design, not code)             | Yes (multi-model code review)                      |
| Automated revert      | No                                           | Yes (automatic revert after 2 failed fix attempts) |

#### 4.2 Error Handling Philosophy

| Aspect               | Forge                                                | Anvil                                                  |
| -------------------- | ---------------------------------------------------- | ------------------------------------------------------ |
| Error categories     | 4 (transient, persistent, security, missing context) | Implicit via verification cascade                      |
| Retry budget         | Explicitly bounded (3×2=6 max)                       | 2 fix attempts per check, 2 adversarial rounds         |
| Non-blocking failure | PostMortem, R-Knowledge                              | Small tasks have reduced verification                  |
| Escalation path      | Agent → orchestrator retry → orchestrator escalation | Fix → re-verify → revert → present with Low confidence |
| Pipeline halt        | R-Security blocker                                   | N/A (single agent, presents to user)                   |

#### 4.3 Communication Medium

| Aspect       | Forge                   | Anvil                          |
| ------------ | ----------------------- | ------------------------------ |
| Inter-agent  | Memory files (Markdown) | N/A (single agent)             |
| Evidence     | Markdown reports        | SQL tables                     |
| Evaluation   | YAML blocks in Markdown | SQL SELECT results             |
| Human-facing | Markdown artifacts      | Evidence Bundle + code changes |

---

### 5. Anti-Patterns Identified

#### 5.1 Excessive Memory Merge Overhead (Forge)

The orchestrator performs memory merge after **every** agent completion, including between implementation sub-waves. With 23+ agent dispatches per pipeline, this generates significant overhead:

- Memory merge requires dispatching a subagent (not direct file write).
- Pruning at three checkpoints adds complexity.
- Invalidation on revision requires tracking which entries to mark.
- In post-mortem data: 32 dispatches, suggesting ~20+ merge operations per run.

#### 5.2 Inconsistent Severity Taxonomies (Forge)

Three different severity vocabularies across clusters:

- CT: Critical / High / Medium / Low
- R: Blocker / Major / Minor
- V-Build: PASS / FAIL
- The orchestrator includes a special case: "If severity value is 'Critical' instead of 'Blocker': treat as Blocker for safety, but flag a prompt compliance gap."

#### 5.3 Agent Count vs. Value (Forge)

23 agent files (including deprecated `critical-thinker.agent.md`). Many share near-identical Operating Rules, Error Handling, and Memory-First Reading sections. The CT cluster replaced a single critical-thinker with 4 specialized agents. Whether 4 focused reviews is better than 1 comprehensive review is an empirical question.

#### 5.4 Prose Where Data Would Be More Reliable (Forge)

Memory files, verification reports, and cluster decisions all use Markdown prose. The orchestrator must parse prose to extract severity levels, status codes, and routing decisions. Contrast with Anvil's SQL-queryable verification ledger.

#### 5.5 Over-Documentation (Forge)

Artifact evaluations produced by every consuming agent (14 evaluators) create a large documentation footprint. In the post-mortem run, 39 evaluation blocks across 16 files were generated. The signal-to-noise ratio depends on how these are used downstream — primarily by PostMortem for metrics.

---

### 6. Notable Reusable Patterns for New System

#### 6.1 Evidence-First Verification (from Anvil)

- SQL-tracked verification with INSERT-before-report rule.
- Evidence Bundle as SELECT, not prose.
- SQL count gates at critical transitions.
- Machine-verifiable verification records.

#### 6.2 Multi-Model Adversarial Review (from Anvil)

- Parallel review by 3 different models with same prompt.
- INSERT each verdict independently.
- Fix-and-re-review cycle with bounded iterations.
- Escalation to 3 reviewers for high-risk files.

#### 6.3 Typed Completion Contracts (from Forge)

- Three-state return (DONE/NEEDS_REVISION/ERROR).
- Deterministic routing based on return state.
- Each state maps to exactly one orchestrator action.

#### 6.4 Cluster Dispatch (from Forge)

- Pattern A/B/C as reusable dispatch templates.
- Concurrency cap with sub-wave partitioning.
- Sequential gate before parallel dispatch (Pattern B).

#### 6.5 Risk-Based Escalation (combined)

- Anvil: Task sizing (S/M/L) × file risk (🟢/🟡/🔴) drives verification depth.
- Forge: Severity ratings from CT/R drive revision routing.
- Combined: Risk classification drives both verification depth and revision routing.

#### 6.6 Baseline Capture (from Anvil)

- Capture pre-change state before modifications.
- Compare before/after for regression detection.
- Not responsible for pre-existing failures, but responsible for not worsening.

#### 6.7 Pushback Pattern (from Anvil)

- Evaluate request quality before execution.
- Structured interaction with explicit choices.
- Both implementation and requirements-level concerns.

#### 6.8 Self-Verification Before Return (from both)

- Every agent checks its own output against input requirements.
- Forge: Designer, Spec, CT agents, Implementer (Medium+), Orchestrator cluster decisions.
- Anvil: Evidence gates, minimum signal requirements, SQL count checks.

#### 6.9 Anti-Drift Anchors (from Forge)

- Identity reminders at end of every agent definition.
- Prevents context window drift in long conversations.

#### 6.10 Non-Blocking Graceful Degradation (from both)

- Forge: PostMortem ERROR and R-Knowledge ERROR are non-blocking.
- Forge: Memory failure is non-blocking.
- Anvil: Pre-existing failures noted but don't block progress.

---

## File References

| File                                                                                                                                                         | Rationale                                                                                                                                                                                                                |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| [.github/agents/orchestrator.agent.md](../../.github/agents/orchestrator.agent.md)                                                                           | Core pipeline orchestration, all dispatch patterns, cluster decision logic, memory lifecycle, tool restrictions, NEEDS_REVISION routing, telemetry pattern                                                               |
| [.github/agents/dispatch-patterns.md](../../.github/agents/dispatch-patterns.md)                                                                             | Formal definitions of Pattern A, B, C dispatch patterns                                                                                                                                                                  |
| [.github/agents/evaluation-schema.md](../../.github/agents/evaluation-schema.md)                                                                             | Centralized artifact evaluation YAML schema, error fallback, collision avoidance rules                                                                                                                                   |
| [.github/agents/researcher.agent.md](../../.github/agents/researcher.agent.md)                                                                               | Memory-first reading pattern, focused research pattern, retrieval strategy                                                                                                                                               |
| [.github/agents/implementer.agent.md](../../.github/agents/implementer.agent.md)                                                                             | TDD workflow, self-reflection pattern, TDD fallback detection, code quality principles                                                                                                                                   |
| [.github/agents/designer.agent.md](../../.github/agents/designer.agent.md)                                                                                   | Self-verification before return, revision mode pattern, design.md contents structure                                                                                                                                     |
| [.github/agents/spec.agent.md](../../.github/agents/spec.agent.md)                                                                                           | Self-verification of testable ACs, feature.md contents structure                                                                                                                                                         |
| [.github/agents/planner.agent.md](../../.github/agents/planner.agent.md)                                                                                     | Task size limits, dependency-aware waves, plan validation, pre-mortem analysis, replan cross-referencing                                                                                                                 |
| [.github/agents/ct-security.agent.md](../../.github/agents/ct-security.agent.md)                                                                             | Adversarial mindset, risk categories, finding severity format                                                                                                                                                            |
| [.github/agents/v-build.agent.md](../../.github/agents/v-build.agent.md)                                                                                     | Binary gate pattern, build system detection, read-only enforcement                                                                                                                                                       |
| [.github/agents/v-tests.agent.md](../../.github/agents/v-tests.agent.md)                                                                                     | Cross-task integration detection, V-Build dependency                                                                                                                                                                     |
| [.github/agents/v-feature.agent.md](../../.github/agents/v-feature.agent.md)                                                                                 | Feature-level acceptance criteria verification, readiness assessment                                                                                                                                                     |
| [.github/agents/r-quality.agent.md](../../.github/agents/r-quality.agent.md)                                                                                 | Review depth tiers, quality standard calibration                                                                                                                                                                         |
| [.github/agents/r-knowledge.agent.md](../../.github/agents/r-knowledge.agent.md)                                                                             | Knowledge evolution, safety constraint filter (KE-SAFE-6), auto-apply pattern, append-only decisions.md                                                                                                                  |
| [.github/agents/post-mortem.agent.md](../../.github/agents/post-mortem.agent.md)                                                                             | Quantitative-only metrics, telemetry consolidation, non-blocking pattern                                                                                                                                                 |
| [.github/agents/documentation-writer.agent.md](../../.github/agents/documentation-writer.agent.md)                                                           | Delta-only accuracy verification, documentation capabilities                                                                                                                                                             |
| [.github/agents/critical-thinker.agent.md](../../.github/agents/critical-thinker.agent.md)                                                                   | Deprecated single CT agent (replaced by CT cluster), risk categories as floor not ceiling                                                                                                                                |
| [Anvil/anvil.agent.md](../../Anvil/anvil.agent.md)                                                                                                           | Full Anvil Loop, task sizing, risk classification, verification cascade, SQL ledger, adversarial review, evidence gating, pushback, baseline capture, confidence scoring, git hygiene, session recall, boost, learn step |
| [.github/prompts/feature-workflow.prompt.md](../../.github/prompts/feature-workflow.prompt.md)                                                               | Feature workflow prompt with variables, key artifacts table, cluster parallelization rules                                                                                                                               |
| [docs/feature/orchestrator-tool-restriction/post-mortems/2025-07-17-post-mortem.md](../orchestrator-tool-restriction/post-mortems/2025-07-17-post-mortem.md) | Empirical data on pattern effectiveness: CT iteration costs, evaluation accuracy scores, recurring issues                                                                                                                |

---

## Assumptions & Limitations

1. **Article principles inferred from initial-request.md only**: The actual blog post was not available as a file in the workspace; analysis relies on the summary in initial-request.md.
2. **Empirical data limited to one run**: Only one post-mortem (orchestrator-tool-restriction, 2025-07-17) was available. Pattern effectiveness conclusions are based on this single data point.
3. **Anvil session recall SQL**: The session_store database referenced by Anvil's Recall step was not available for inspection — patterns described from the agent definition only.
4. **No runtime execution data for Anvil**: Unlike Forge (which has post-mortem telemetry), there's no execution trace for Anvil to compare actual behavior against defined patterns.

---

## Open Questions

1. **Memory merge cost quantification**: How much context window budget does memory merging consume per pipeline run? Is there a measurable impact on downstream agent quality?
2. **CT cluster iteration frequency**: The post-mortem showed 3 CT iterations in one run — is this typical? Does the CT cluster frequently find issues, or was this an outlier?
3. **Anvil SQL ledger persistence**: Is the SQLite database ephemeral (per-session) or persistent? How does it interact with VS Code's session management?
4. **Evaluation schema ROI**: Do the 14-agent artifact evaluations provide sufficient signal for pipeline improvement? The post-mortem used them effectively, but at what documentation cost?
5. **Adversarial review model availability**: The three models specified (gpt-5.3-codex, gemini-3-pro-preview, claude-opus-4.6) — are all reliably available in the target environment? What's the fallback if a model is unavailable?

---

## Research Metadata

- **confidence_level**: high — both systems were fully examined with all agent definitions read
- **coverage_estimate**: ~95% of pattern-relevant content examined. All 23 Forge agent files plus Anvil agent definition, dispatch-patterns.md, evaluation-schema.md, feature-workflow prompt, and available post-mortem data were read.
- **gaps**: (1) No access to article blog post full text — only initial-request.md summary. Impact: failure prevention patterns analysis may miss nuances. (2) No Anvil execution traces available — pattern analysis is definition-only, not empirically validated. (3) Only one Forge post-mortem available — limits confidence in frequency-based pattern conclusions.
