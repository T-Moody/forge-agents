# Dependencies Research — Agent Improvements

## Focus Area: dependencies

## Summary

Forge agents form a strict linear data-flow chain via markdown artifact files with well-defined I/O contracts; adopting proposed improvements creates 3 tightly-coupled change clusters (TDD requires coordinated verifier+orchestrator updates, per-task agent routing requires planner+orchestrator updates, security review depth may require planner metadata additions) plus 6 independently-adoptable improvements and several cross-cutting consistency gaps that should be addressed uniformly.

---

## 1. Current Forge Inter-Agent Data Flow

### 1.1 Data Flow Diagram

```
User Request
    │
    ▼
┌──────────────┐  creates    ┌──────────────────┐
│ Orchestrator │────────────▶│ initial-request.md│
└──────┬───────┘             └──────────────────┘
       │ dispatches ×3 concurrently
       ▼
┌──────────────┐  writes     ┌─────────────────────────────────┐
│ Researcher   │────────────▶│ research/architecture.md        │
│ (focused ×3) │────────────▶│ research/impact.md              │
│              │────────────▶│ research/dependencies.md        │
└──────┬───────┘             └─────────────────────────────────┘
       │ reads: initial-request.md, codebase
       ▼
┌──────────────┐  writes     ┌──────────────────┐
│ Researcher   │────────────▶│ analysis.md      │
│ (synthesis)  │             └──────────────────┘
└──────┬───────┘  reads: initial-request.md, research/*.md
       ▼
┌──────────────┐  writes     ┌──────────────────┐
│ Spec         │────────────▶│ feature.md       │
└──────┬───────┘             └──────────────────┘
       │ reads: initial-request.md, analysis.md
       ▼
┌──────────────┐  writes     ┌──────────────────┐
│ Designer     │────────────▶│ design.md        │
└──────┬───────┘             └──────────────────┘
       │ reads: initial-request.md, analysis.md, feature.md
       ▼
┌──────────────────┐  writes ┌──────────────────────────────┐
│ Critical Thinker │────────▶│ design_critical_review.md    │
└──────┬───────────┘         └──────────────────────────────┘
       │ reads: design.md
       │ (if issues found → loop back to Designer, max 1 loop)
       ▼
┌──────────────┐  writes     ┌──────────────────┐  ┌──────────────────┐
│ Planner      │────────────▶│ plan.md          │  │ tasks/*.md       │
└──────┬───────┘             └──────────────────┘  └──────────────────┘
       │ reads: initial-request.md, analysis.md, feature.md, design.md
       ▼
┌──────────────┐  writes     code files + updates task file
│ Implementer  │  (one per task, parallel within wave)
│ (×N per wave)│
└──────┬───────┘  reads: tasks/<task>.md, feature.md, design.md
       │          MUST NOT read: plan.md
       ▼
┌──────────────┐  writes     ┌──────────────────┐
│ Verifier     │────────────▶│ verifier.md      │
└──────┬───────┘             └──────────────────┘
       │ reads: initial-request.md, feature.md, design.md,
       │        plan.md, tasks/*.md, codebase, (prev verifier.md)
       │ (if ERROR → loop: planner re-plans failing tasks → re-implement → re-verify, max 3×)
       ▼
┌──────────────┐  writes     ┌──────────────────┐
│ Reviewer     │────────────▶│ review.md        │
└──────┬───────┘             └──────────────────┘
       │ reads: initial-request.md, git diff, codebase
       │ also updates: .github/instructions
       │ (if ERROR → loop back to Planner)
       ▼
     DONE
```

### 1.2 Exact Agent Input/Output Table

| Agent                      | Inputs (files read)                                                                           | Outputs (files written)                 | Completion Contract                                                                                          |
| -------------------------- | --------------------------------------------------------------------------------------------- | --------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| **Orchestrator**           | plan.md (wave parsing), verifier.md (failure routing), review.md (blocking routing)           | initial-request.md                      | N/A (top-level coordinator)                                                                                  |
| **Researcher (focused)**   | initial-request.md, codebase                                                                  | `research/<focus-area>.md`              | `DONE: <focus-area> — <summary>` / `ERROR: <reason>`                                                         |
| **Researcher (synthesis)** | initial-request.md, `research/*.md`                                                           | analysis.md                             | `DONE: synthesis — <summary>` / `ERROR: <reason>`                                                            |
| **Spec**                   | initial-request.md, analysis.md                                                               | feature.md                              | `DONE: <summary>` / `ERROR: <reason>`                                                                        |
| **Designer**               | initial-request.md, analysis.md, feature.md                                                   | design.md                               | `DONE: <summary>` / `ERROR: <reason>`                                                                        |
| **Critical Thinker**       | design.md                                                                                     | design_critical_review.md               | _(Not explicitly defined in agent file — see Open Questions)_                                                |
| **Planner**                | initial-request.md, analysis.md, feature.md, design.md; (replan: verifier.md or review.md)    | plan.md, tasks/\*.md                    | `DONE: <N tasks>, <M waves>` / `ERROR: <reason>`                                                             |
| **Implementer**            | tasks/\<task\>.md, feature.md, design.md                                                      | code files, task file updates           | `DONE: <task-id>` / `ERROR: <reason>`                                                                        |
| **Verifier**               | initial-request.md, feature.md, design.md, plan.md, tasks/\*.md, codebase, (prev verifier.md) | verifier.md                             | `DONE: verification passed (<N> tasks, <M> tests)` / `ERROR: <summary> (<N> build, <M> failures, <K> tasks)` |
| **Reviewer**               | initial-request.md, git diff, codebase                                                        | review.md, .github/instructions updates | `DONE: review complete` / `ERROR: <blocking concerns>`                                                       |

### 1.3 Key Observations About Forge Data Flow

1. **Linear artifact chain**: Each agent reads artifacts from prior stages and writes exactly one primary output. The orchestrator is the sole router.
2. **Explicit file passing**: The orchestrator tells each agent exactly which files to read. Agents do not discover their inputs.
3. **Implementer isolation**: Implementers are deliberately firewalled from plan.md to prevent scope creep.
4. **Verifier reads everything**: The verifier has the broadest read scope of any agent (initial-request, feature, design, plan, tasks, codebase, previous verifier report).
5. **Two feedback loops**: Verifier→Planner (max 3×) and Reviewer→Planner (once implied, no explicit max).
6. **No shared state**: No agent writes to a shared state file. All inter-agent communication is through distinct artifact files.

---

## 2. Gem Team Inter-Agent Data Flow

### 2.1 Data Flow Diagram

```
User Request
    │
    ▼
┌──────────────────┐  generates plan_id
│ gem-orchestrator  │
└──────┬───────────┘
       │ dispatches ×N concurrently
       ▼
┌──────────────────┐  writes   ┌───────────────────────────────────────────────┐
│ gem-researcher   │──────────▶│ docs/plan/{plan_id}/research_findings_*.yaml │
│ (×N by focus)    │           └───────────────────────────────────────────────┘
└──────┬───────────┘  reads: codebase, memories
       │
       ▼ PAUSE: findings review (plan_review)
       │
┌──────────────────┐  writes   ┌─────────────────────────────────┐
│ gem-planner      │──────────▶│ docs/plan/{plan_id}/plan.yaml  │
└──────┬───────────┘           └─────────────────────────────────┘
       │ reads: research_findings*.yaml
       │ detects mode: initial | replan | extension
       │
       ▼ PAUSE: plan approval (plan_review)
       │
┌──────────────────┐  reads plan.yaml, dispatches up to 4 pending tasks
│ gem-orchestrator  │──────────────────────────────────────────────┐
└──────┬───────────┘                                               │
       ▼                                                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│ gem-implementer / gem-chrome-tester / gem-devops /                 │
│ gem-reviewer / gem-documentation-writer                            │
│ (each reads plan.yaml for task context, up to 4 concurrent)       │
└──────┬──────────────────────────────────────────────────────────────┘
       │ returns JSON: {status, task_id, summary}
       ▼
┌──────────────────┐  updates plan.yaml task statuses
│ gem-orchestrator  │
└──────┬───────────┘
       │ if failed → routes to gem-planner (replan) or gem-implementer (fix)
       │ if requires_review → routes to gem-reviewer
       │ PAUSE: batch confirmation
       │ loops until all tasks completed
       ▼
     DONE (walkthrough_review)
```

### 2.2 Key Differences in Data Flow

| Aspect                | Forge                                              | Gem Team                                                                       |
| --------------------- | -------------------------------------------------- | ------------------------------------------------------------------------------ |
| **State management**  | N separate markdown files, each owned by one agent | Single `plan.yaml` file, read/written by multiple agents                       |
| **Context discovery** | Orchestrator explicitly passes file paths          | Agents read plan.yaml directly using task_id + plan_id                         |
| **Contract format**   | Text: `DONE: <summary>` / `ERROR: <reason>`        | JSON: `{status, task_id/plan_id, summary}`                                     |
| **Task routing**      | All impl tasks → implementer                       | Per-task agent field: implementer, chrome-tester, devops, reviewer, doc-writer |
| **Feedback routing**  | Orchestrator parses DONE/ERROR text                | Orchestrator reads JSON status field                                           |
| **Verification**      | Separate verifier agent post-implementation        | Inline per-task via `task_block.verification` commands                         |
| **Research output**   | Markdown partials → synthesis step → analysis.md   | YAML findings files → direct to planner (no synthesis)                         |
| **Memory**            | None (artifact files only)                         | Cross-agent memory with citations                                              |

---

## 3. Change Propagation Analysis

### 3.1 Change Dependency Matrix

Each proposed improvement is analyzed for which other agents/files MUST change if it is adopted.

| #   | Improvement             | Primary Target                                     | Must Also Change                              | May Optionally Change                                        | Independent?                           |
| --- | ----------------------- | -------------------------------------------------- | --------------------------------------------- | ------------------------------------------------------------ | -------------------------------------- |
| 1   | Pre-mortem analysis     | planner.agent.md                                   | —                                             | —                                                            | **YES**                                |
| 2   | TDD enforcement         | implementer.agent.md                               | verifier.agent.md, orchestrator.agent.md      | —                                                            | **NO**                                 |
| 3   | Task size limits        | planner.agent.md                                   | —                                             | —                                                            | **YES**                                |
| 4   | Concurrency cap         | orchestrator.agent.md                              | —                                             | —                                                            | **YES**                                |
| 5   | Security-focused review | reviewer.agent.md                                  | planner.agent.md (if tiered by task metadata) | orchestrator.agent.md (if passing task metadata to reviewer) | **CONDITIONAL**                        |
| 6   | Hybrid retrieval        | researcher.agent.md                                | —                                             | —                                                            | **YES**                                |
| 7   | Per-task agent routing  | planner.agent.md + orchestrator.agent.md           | —                                             | —                                                            | **NO** (must be coordinated)           |
| 8   | Cross-agent memory      | researcher.agent.md + reviewer.agent.md            | —                                             | all other agents                                             | **YES** (minimal viable with 2 agents) |
| 9   | Human approval gates    | orchestrator.agent.md + feature-workflow.prompt.md | —                                             | —                                                            | **NO** (must be coordinated)           |
| 10  | Self-reflection         | implementer.agent.md + reviewer.agent.md           | —                                             | —                                                            | **YES** (each agent independent)       |

### 3.2 Detailed Change Propagation

#### Cluster A: TDD Enforcement (Improvement #2) — TIGHTLY COUPLED

This is the most impactful cross-agent change. Three files must change in coordination:

1. **implementer.agent.md**: Add TDD workflow (write failing tests → write code → confirm pass → run `get_errors`)
2. **verifier.agent.md**: Must change role from "sole owner of build and test execution" to "integration-level verifier." Current text explicitly states: _"The verifier is the **sole agent responsible for building the project and running tests**. Implementer agents write code and tests but never execute them."_ This entire framing must be rewritten.
3. **orchestrator.agent.md**: Global rule states _"Implementers never build or run tests. All build and test execution is the verifier's responsibility."_ Must be rewritten to: "Implementers perform unit-level TDD. The verifier performs integration-level verification."

**If implementer gets TDD but verifier/orchestrator are NOT updated**: The verifier's instructions would contradict reality (it claims implementers never run tests, but they do). The orchestrator's global rule would be false. The verifier might redundantly re-verify already-passing unit tests, which is acceptable but inefficient. No hard breakage, but significant inconsistency.

#### Cluster B: Per-Task Agent Routing (Improvement #7) — TIGHTLY COUPLED

Two files must change together:

1. **planner.agent.md**: Must add `agent` field to task file requirements (with default: `implementer`)
2. **orchestrator.agent.md**: Must read `agent` field from task files in Step 5.2 and dispatch to the correct agent instead of always dispatching `implementer`

**If only planner changes**: Task files would have an `agent` field that nobody reads. Harmless but useless.
**If only orchestrator changes**: Orchestrator would look for an `agent` field that doesn't exist and must fall back to `implementer`. No breakage if default is handled.

#### Cluster C: Security Review with Tiered Depth (Improvement #5) — CONDITIONALLY COUPLED

Two implementation approaches with different dependency profiles:

**Approach A — Reviewer self-determines depth (NO planner dependency)**:

- Reviewer examines the git diff and codebase to determine if changes touch auth, PII, payments, etc.
- Review depth is derived from what was changed, not from task metadata.
- Only reviewer.agent.md changes. Planner and orchestrator are unaffected.

**Approach B — Reviewer reads task metadata (PLANNER dependency)**:

- Planner flags tasks with `security_sensitive: true/false` and `priority: high/medium/low`.
- Reviewer uses these fields to determine review depth.
- Requires: planner.agent.md (add fields), task file format (add fields), possibly orchestrator.agent.md (pass task metadata to reviewer).
- Currently the reviewer does NOT receive task files — it reads git diff and codebase. The orchestrator would need to pass task metadata or the reviewer would need to read plan.md/tasks/.

**Recommendation**: Approach A is simpler and avoids coupling. The reviewer can grep the diff for auth/security keywords to determine depth.

#### Independent Changes (No Cross-Agent Dependencies)

- **#1 Pre-mortem**: Planner writes extra section in plan.md. No agent parses the pre-mortem section — it's informational for humans and the planner itself. The orchestrator only parses execution waves from plan.md. No impact.
- **#3 Task size limits**: Planner internal constraint. May produce more tasks/waves, but the format is unchanged. Orchestrator's wave parsing is unaffected.
- **#4 Concurrency cap**: Orchestrator internal behavior. Splits large waves into sub-waves. No format changes. No agent impact.
- **#6 Hybrid retrieval**: Researcher internal methodology. No output format change.
- **#10 Self-reflection**: Internal step within each agent. No output format or contract change.

---

## 4. Cross-Cutting Concerns Inventory

These are patterns that Gem Team applies consistently across ALL agents but Forge either lacks entirely or applies inconsistently.

### 4.1 Communication Style Rules

**Gem Team** (identical in ALL 8 agents):

> "Communication: Output ONLY the requested deliverable. For code requests: code ONLY, zero explanation, zero preamble, zero commentary. For questions: direct answer in ≤3 sentences. Never explain your process unless explicitly asked 'explain how'."

**Forge**: No consistent communication rules across agents. Each agent has different verbosity levels. Some agents (spec, designer) have detailed output format requirements but no communication style guidance.

**Impact on improvements**: If adding communication rules, all 9 agents plus the orchestrator must be updated for consistency.

### 4.2 Tool Usage Patterns

**Gem Team** (identical in ALL 8 agents):

> "Context-efficient file reading: prefer semantic search, file outlines, and targeted line-range reads; limit to 200 lines per read. Built-in preferred; batch independent calls. Prefer multi_replace_string_in_file for file edits (batch for efficiency)."

**Forge**: No tool usage guidance in any agent. Only the researcher mentions retrieval approaches (and even then, not prescriptively).

**Impact on improvements**: If adding hybrid retrieval (#6) to researcher, consider adding general tool usage rules to ALL agents for consistency.

### 4.3 Error Handling Conventions

**Gem Team** (consistent pattern across agents):

> "Handle errors: transient→handle, persistent→escalate"

- Plus agent-specific rules: security issues→halt, missing context→blocked, etc.

**Forge**: No error handling guidance beyond the DONE/ERROR contract. Agents don't know when to retry vs. escalate.

### 4.4 File Reading Efficiency Rules

**Gem Team**: "limit to 200 lines per read"
**Forge**: No guidance. Agents may read entire files or make many small reads.

### 4.5 Context Passing Conventions

**Forge**: Orchestrator explicitly passes file paths to each agent. Each agent has a strict `## Inputs` section listing what it reads.
**Gem Team**: Agents receive `plan_id` and `task_id` and read `plan.yaml` directly for full context.

**Forge's approach is more explicit and debuggable** — you can trace exactly what each agent sees. Gem's approach is more flexible but introduces shared state coupling.

### 4.6 Completion Contract Format

**Forge**: Text-based one-line format — `DONE: <details>` / `ERROR: <details>`

- Each agent has a different detail format (e.g., planner returns task count, verifier returns test counts)
- Orchestrator parses these text strings to route decisions

**Gem Team**: JSON — `{"status": "success|failed|needs_revision", "plan_id/task_id": "...", "summary": "..."}`

- Uniform structure across all agents
- Machine-parseable

**If contract format changes**: This is the highest-risk cross-cutting change. EVERY agent file and the orchestrator's parsing logic would need simultaneous updates. The existing `DONE:`/`ERROR:` text parsing in the orchestrator would break.

---

## 5. Implementation Ordering Constraints

### 5.1 Dependency Graph of Improvements

```
Independent (can be done in any order, at any time):
  ├── #1  Pre-mortem analysis (planner only)
  ├── #3  Task size limits (planner only)
  ├── #4  Concurrency cap (orchestrator only)
  ├── #6  Hybrid retrieval (researcher only)
  └── #10 Self-reflection (implementer + reviewer, each independent)

Cluster A — Must be coordinated:
  #2 TDD enforcement
    ├── implementer.agent.md  (primary change)
    ├── verifier.agent.md     (role redefinition — MUST update)
    └── orchestrator.agent.md (global rule update — MUST update)

Cluster B — Must be coordinated:
  #7 Per-task agent routing
    ├── planner.agent.md      (add agent field to task format)
    └── orchestrator.agent.md (read agent field, dispatch accordingly)

Cluster C — Must be coordinated:
  #9 Human approval gates
    ├── orchestrator.agent.md           (add conditional pause logic)
    └── feature-workflow.prompt.md      (add APPROVAL_MODE variable)

Conditionally coupled:
  #5 Security review
    ├── reviewer.agent.md (always required)
    └── planner.agent.md  (only if tiered depth uses task metadata)

Sequential dependency:
  #8 Cross-agent memory
    └── Must define convention FIRST (docs/decisions.md)
    └── Then update researcher.agent.md (read)
    └── Then update reviewer.agent.md (write)
```

### 5.2 Recommended Implementation Order

**Phase 1 — Independent, High-Impact (P0)**

1. Pre-mortem analysis (#1) — planner only, no dependencies
2. TDD enforcement (#2) — **must update implementer + verifier + orchestrator together**
3. Hybrid retrieval (#6) — researcher only, no dependencies

**Phase 2 — Independent, Medium-Impact (P1)** 4. Task size limits (#3) — planner only 5. Concurrency cap (#4) — orchestrator only 6. Security review (#5) — reviewer only (Approach A: self-determined depth)

**Phase 3 — Coupled Changes (P2)** 7. Per-task agent routing (#7) — planner + orchestrator together 8. Human approval gates (#9) — orchestrator + prompt together 9. Cross-agent memory (#8) — convention first, then researcher + reviewer

**Phase 4 — Polish (P3)** 10. Self-reflection (#10) — implementer + reviewer (independent of each other)

### 5.3 Ordering Constraints (MUST-BEFORE relationships)

- **#2 (TDD) BEFORE #10 (self-reflection in implementer)**: If implementer gets both TDD and self-reflection, implement TDD first so self-reflection can reference the TDD cycle.
- **#3 (task size limits) BEFORE #7 (per-task routing)**: Smaller tasks make per-task agent routing more meaningful. If tasks are too large, routing them to specialized agents is less effective.
- **#1 (pre-mortem) and #3 (task size limits) can be done together**: Both are planner-only changes and affect different sections of the planner. No conflict.
- **#4 (concurrency cap) and #7 (per-task routing) can be done together**: Both change the orchestrator's dispatch logic. Concurrency cap applies to sub-waves; per-task routing applies to agent selection within a wave. No conflict, but doing both simultaneously requires careful orchestrator editing.

---

## 6. Breaking Change Analysis

### 6.1 Changes That Break Current Contracts

| Change                                  | What Breaks                                                                    | Severity                                                            | Mitigation                                                                   |
| --------------------------------------- | ------------------------------------------------------------------------------ | ------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| **#2 TDD enforcement**                  | Orchestrator global rule "Implementers never build or run tests" becomes false | **HIGH** — orchestrator could enforce wrong behavior if not updated | Update orchestrator.agent.md and verifier.agent.md simultaneously            |
| **#2 TDD enforcement**                  | Verifier "sole owner" identity invalidated                                     | **MEDIUM** — verifier might waste time re-running tests already run | Redefine verifier as "integration-level verifier"                            |
| **#7 Per-task agent routing**           | Orchestrator Step 5.2 assumes all tasks go to `implementer`                    | **HIGH** — non-implementer tasks would be sent to wrong agent       | Update orchestrator.agent.md dispatch logic                                  |
| **Contract format change** (if adopted) | Orchestrator parses `DONE:`/`ERROR:` text; JSON would not match                | **CRITICAL** — entire pipeline breaks                               | Don't change contract format, OR change all agents + orchestrator atomically |

### 6.2 Changes That Are Backward Compatible

| Change                    | Why Safe                                                             |
| ------------------------- | -------------------------------------------------------------------- |
| **#1 Pre-mortem**         | Adds new section to plan.md; no agent parses it                      |
| **#3 Task size limits**   | Internal planner constraint; task format unchanged                   |
| **#4 Concurrency cap**    | Internal orchestrator behavior; no format changes                    |
| **#5 Security review**    | Reviewer adds new checks; output format (review.md) unchanged        |
| **#6 Hybrid retrieval**   | Internal methodology; output format (research partials) unchanged    |
| **#8 Cross-agent memory** | Additive — new file convention; existing agents don't need to change |
| **#9 Approval gates**     | Optional, gated by prompt variable; default behavior unchanged       |
| **#10 Self-reflection**   | Internal step; completion contract unchanged                         |

### 6.3 Specific Contract Dependency Analysis

The `DONE:`/`ERROR:` contract is the critical integration point. The orchestrator's behavior depends on parsing these strings:

- **Orchestrator waits for `DONE:` or `ERROR:`** from every subagent (stated in Global Rules).
- **On `ERROR:` from verifier**: Reads `verifier.md` → routes to planner for re-planning.
- **On `ERROR:` from reviewer**: Routes back to planning.
- **On `ERROR:` from any other agent**: Retries the step.

**If any agent changes its contract format** (e.g., to JSON), the orchestrator's text parsing breaks. The orchestrator doesn't use regex — it looks for the `DONE:` or `ERROR:` prefix.

**Current gap**: The critical-thinker agent file has NO explicit completion contract section. The orchestrator expects `DONE:` or `ERROR:` from it (Step 3b: "Wait for DONE: or ERROR: with a one-line summary before proceeding"), but the critical-thinker doesn't define this contract. This is a pre-existing bug that should be fixed regardless of improvements.

### 6.4 Verifier Hardcoded Commands

The verifier currently hardcodes `dotnet build TourneyPal.sln` and `dotnet test tests/TourneyPal.Tests`. Any improvement effort should parameterize these or note that the verifier needs project-specific configuration. This is not a breaking change from the improvements, but it's a pre-existing coupling that limits portability.

---

## 7. Agent Contract Dependencies Map

```
Orchestrator
├── expects DONE:/ERROR: from ALL agents
├── parses verifier.md on ERROR (routes to planner)
├── parses review.md on ERROR (routes to planner)
├── reads plan.md for wave structure
│
├── Researcher (focused) ──► DONE: <focus-area> — <summary>
│   └── writes: research/<focus-area>.md
│
├── Researcher (synthesis) ──► DONE: synthesis — <summary>
│   └── writes: analysis.md
│   └── reads: research/*.md
│
├── Spec ──► DONE: <summary>
│   └── writes: feature.md
│   └── reads: analysis.md
│
├── Designer ──► DONE: <summary>
│   └── writes: design.md
│   └── reads: analysis.md, feature.md
│
├── Critical Thinker ──► DONE: <summary> (NOT DEFINED IN AGENT FILE)
│   └── writes: design_critical_review.md
│   └── reads: design.md
│
├── Planner ──► DONE: <N tasks>, <M waves>
│   └── writes: plan.md, tasks/*.md
│   └── reads: analysis.md, feature.md, design.md
│   └── (replan reads: verifier.md or review.md)
│
├── Implementer (×N) ──► DONE: <task-id>
│   └── writes: code files, updates task file
│   └── reads: tasks/<task>.md, feature.md, design.md
│   └── MUST NOT read: plan.md
│
├── Verifier ──► DONE: verification passed (...) / ERROR: <summary> (...)
│   └── writes: verifier.md
│   └── reads: initial-request.md, feature.md, design.md, plan.md, tasks/*.md, codebase
│
└── Reviewer ──► DONE: review complete / ERROR: <blocking concerns>
    └── writes: review.md, .github/instructions updates
    └── reads: initial-request.md, git diff, codebase
```

---

## 8. File References

| File                                         | Relevance                                                                                                                            |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `.github/agents/orchestrator.agent.md`       | Central coordinator; owns all routing logic, global rules, wave dispatch. Any change to contracts or dispatch must update this file. |
| `.github/agents/researcher.agent.md`         | Hybrid retrieval (#6) and cross-agent memory (#8) changes target this file.                                                          |
| `.github/agents/spec.agent.md`               | No proposed changes. Independent of all improvements.                                                                                |
| `.github/agents/designer.agent.md`           | No proposed changes. Independent of all improvements.                                                                                |
| `.github/agents/critical-thinker.agent.md`   | Pre-existing gap: missing completion contract. Should be fixed as part of any improvement effort.                                    |
| `.github/agents/planner.agent.md`            | Target for #1 (pre-mortem), #3 (task size limits), #7 (per-task routing), and possibly #5 (security metadata).                       |
| `.github/agents/implementer.agent.md`        | Target for #2 (TDD) and #10 (self-reflection).                                                                                       |
| `.github/agents/verifier.agent.md`           | Must update for #2 (TDD) — role redefinition from "sole owner" to "integration-level verifier." Also has hardcoded dotnet commands.  |
| `.github/agents/reviewer.agent.md`           | Target for #5 (security review), #8 (memory/decision log), #10 (self-reflection).                                                    |
| `.github/prompts/feature-workflow.prompt.md` | Target for #9 (approval gates). Must add APPROVAL_MODE variable.                                                                     |
| `docs/comparison-forge-vs-gem-team.md`       | Reference document. Contains accurate comparison but should not drive changes blindly.                                               |
| `docs/optimization-from-gem-team.md`         | Reference document. Contains implementation guidance for each improvement.                                                           |

---

## 9. Assumptions & Limitations

### Assumptions

1. **Output directory constraint**: All improved agents are written to `NewAgentsAndPrompts/`, not modifying `.github/agents/`. This means all agents must be self-consistent within the new directory — no mixing of old and new agents.
2. **Contract format preserved**: The `DONE:`/`ERROR:` text contract format will be maintained. Switching to JSON would require changing all 9 agents + the orchestrator simultaneously.
3. **No new agents in core pipeline**: Chrome-tester, devops, and documentation-writer are not added to the core pipeline. Per-task routing (#7) is future-proofed but only routes to `implementer` by default.
4. **Verifier remains a separate agent**: TDD enforcement supplements but does not replace the verifier. The verifier continues to perform integration-level verification.
5. **Markdown artifact format preserved**: No switch to YAML state files. All agents continue to use markdown artifacts.
6. **Fixed pipeline preserved**: The pipeline remains deterministic (setup → research → spec → design → review → plan → implement → verify → review). No dynamic DAG dispatch.

### Limitations

1. **Could not verify Gem Team memory system details**: The memory system references (`/memories/memory-system-patterns.md`, `memory create/update` tools) appear to be Gem-specific infrastructure not available in standard VS Code / Copilot. The cross-agent memory improvement (#8) may need to use a simpler file-based approach.
2. **Gem Team tool activations**: Gem agents reference tools like `activate_vs_code_interaction`, `activate_web_interaction`, `plan_review`, `walkthrough_review`, `manage_todos`, `mcp_sequential-th_sequentialthinking` — these are specific MCP tools or extensions that may not be available. Tool-specific references should not be blindly copied into Forge agents.
3. **Verifier commands are project-specific**: The verifier hardcodes `dotnet build TourneyPal.sln` and `dotnet test tests/TourneyPal.Tests`. This is a pre-existing limitation, not introduced by improvements, but affects portability.

---

## 10. Open Questions

1. **Critical-thinker completion contract**: The critical-thinker agent file has no `## Completion Contract` section. The orchestrator expects `DONE:`/`ERROR:` from it. Should this be added as a prerequisite fix before other improvements?

2. **Security review depth approach**: Should the reviewer determine review depth from the diff content (Approach A — no planner dependency) or from task metadata (Approach B — planner must add `security_sensitive` field)? This determines whether improvement #5 is independent or coupled.

3. **Verifier role after TDD**: If implementers now run unit tests, should the verifier still run the full test suite (including unit tests already passed by implementers), or only integration/e2e tests? Running all tests is safer but slower. Running only integration tests requires the verifier to know which tests are unit vs. integration.

4. **Cross-agent memory mechanism**: Does the target environment support any memory/persistence MCP tools, or must memory be purely file-based (e.g., `docs/decisions.md`)?

5. **Concurrency cap value**: The optimization doc suggests max 4 (matching Gem Team). Is this appropriate for the target environment, or should it be configurable?

6. **Task size limits calibration**: Gem Team uses max 3 files, max 500 lines, max 2 dependencies. Are these limits appropriate for the types of projects Forge is used for, or should they be adjusted?

7. **Approval gates implementation**: How should the `{{APPROVAL_MODE}}` prompt variable be passed in VS Code Copilot's prompt system? Does the prompt template support conditional logic natively?

8. **Self-reflection scope**: Should self-reflection be added to ALL agents (spec, designer, planner, etc.) or only to implementer and reviewer as currently proposed?
