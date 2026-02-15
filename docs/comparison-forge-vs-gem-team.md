# Forge vs. Gem Team — Full Comparison

A detailed, agent-by-agent comparison between **Forge** (this repository) and **[Gem Team](https://github.com/mubaidr/gem-team)** — two multi-agent orchestration frameworks for GitHub Copilot.

---

## Executive Summary

Both frameworks decompose complex development work into specialized agents coordinated by an orchestrator. They share the same foundational insight — single-agent AI coding assistants hit context and quality limits on non-trivial projects — and arrive at structurally similar solutions. The key differences are in agent count, workflow philosophy, state management, and where human approval gates sit.

| Dimension | Forge | Gem Team |
|---|---|---|
| **Agent count** | 9 | 8 |
| **Orchestrator model** | Deterministic fixed pipeline | Dynamic DAG-driven dispatcher |
| **State format** | Markdown artifacts (per-stage files) | YAML state file (`plan.yaml`) |
| **Human gates** | None (fully autonomous) | Mandatory pauses (findings review, plan approval, batch confirmation) |
| **Unique agents** | `spec`, `designer`, `critical-thinker` | `chrome-tester`, `devops`, `documentation-writer` |
| **Parallel execution** | Wave-based (unlimited per wave) | DAG-based (capped at 4 concurrent) |
| **Verification model** | Separate verifier agent (post-implementation) | Inline per-task verification commands |
| **Error recovery** | Re-plan + re-implement failing tasks only (max 3 retries) | Replan or fix via orchestrator routing |

---

## Agent-by-Agent Comparison

### 1. Orchestrator

| Aspect | Forge (`orchestrator`) | Gem Team (`gem-orchestrator`) |
|---|---|---|
| **Core approach** | Fixed 8-step pipeline (setup → research → spec → design → review → plan → implement → verify → review) | Dynamic dispatch loop driven by `plan.yaml` task states |
| **Delegation** | `runSubagent` with explicit step-by-step instructions | `runSubagent` with task_id + plan_id; agent reads `plan.yaml` |
| **Code execution** | Never executes code | `disable-model-invocation: true` — delegates only |
| **Plan format** | `plan.md` (markdown with execution waves) | `plan.yaml` (YAML with DAG, statuses, metadata) |
| **Concurrency cap** | No explicit cap (wave-based, all tasks in wave run concurrently) | Max 4 concurrent agents |
| **Human interaction** | No pause points — fully autonomous | Mandatory pauses: findings review, plan approval, batch confirmation |
| **Memory system** | None — agents communicate through artifact files | Cross-agent memory with citations and just-in-time verification |
| **Error handling** | Retry failed steps; re-plan failing tasks via verifier report (max 3 loops) | Failure routing to planner for replan; revision routing to implementer |

**Analysis:**
Forge's fixed pipeline is more predictable and easier to reason about — you always know what step comes next. Gem Team's dynamic approach is more flexible for complex projects where the workflow may need to adapt mid-execution. Gem Team's mandatory human approval gates add safety but reduce autonomy. Forge trades that for speed by running fully unattended.

---

### 2. Researcher

| Aspect | Forge (`researcher`) | Gem Team (`gem-researcher`) |
|---|---|---|
| **Parallel research** | 3 focused instances (architecture, impact, dependencies) | Multiple instances by focus_area (orchestrator decides count and areas) |
| **Synthesis** | Dedicated synthesis mode — separate invocation merges all partials into `analysis.md` | No explicit synthesis step; orchestrator presents consolidated findings |
| **Output format** | Markdown (`research/<focus-area>.md` + `analysis.md`) | YAML (`research_findings_{focus_area}.yaml`) |
| **Retrieval strategy** | Not specified — agent decides approach | Hybrid retrieval: semantic_search first → grep_search for exact patterns → merge + deduplicate |
| **Confidence tracking** | Assumptions & Limitations section | Formal `research_metadata` with confidence level, coverage %, and gap impact |
| **Memory integration** | None | Reads existing memories before exploration; uses memory system for project context |
| **Scope restrictions** | Focus on assigned area only; no solutioning | Domain-scoped; skip inapplicable YAML sections |
| **External search** | Not restricted | `tavily_search` only for external/framework docs |

**Analysis:**
Gem Team's researcher is more rigorous — it has a formal YAML schema, hybrid retrieval strategy, and confidence metadata. Forge's researcher is simpler and relies on a separate synthesis step to merge findings, which provides a cleaner separation between gathering and consolidating. Forge's explicit synthesis agent is a strength — it ensures findings are deduplicated and organized before downstream agents consume them.

---

### 3. Planner

| Aspect | Forge (`planner`) | Gem Team (`gem-planner`) |
|---|---|---|
| **Input** | Reads `analysis.md`, `feature.md`, `design.md` | Reads all `research_findings*.yaml` files; detects mode (initial/replan/extension) |
| **Output** | `plan.md` (markdown) + individual `tasks/*.md` files | `plan.yaml` (single YAML file with full task DAG) |
| **Task structure** | Individual markdown files per task with goal, depends_on, acceptance criteria, test requirements | YAML task blocks within `plan.yaml` with agent assignment, verification commands, failure modes |
| **Dependency model** | Execution waves — groups of parallel tasks; task-level `depends_on` fields | Full DAG with sequential task IDs and dependency lists |
| **Pre-mortem analysis** | Not included | Built-in — identifies failure scenarios with likelihood, impact, mitigation |
| **Human approval** | Not included — planner returns to orchestrator directly | Mandatory `plan_review` pause; iterates on feedback until approved |
| **Task size limits** | Not specified | Enforced: max 3 files, max 2 dependencies, max 500 lines, max medium effort |
| **Agent assignment** | All implementation tasks go to `implementer` | Per-task agent assignment (implementer, chrome-tester, devops, reviewer, doc-writer) |
| **Replan support** | Via verifier report — only failing tasks re-planned | Mode detection: initial, replan, or extension |
| **Implementation spec** | Not included in plan | `implementation_specification` section with code structure, affected areas, component details |

**Analysis:**
Gem Team's planner is substantially more sophisticated — pre-mortem analysis, task size limits, per-task agent routing, and a formal YAML state file that supports mode detection for replanning. Forge's planner is simpler but produces better-scoped task files (individual markdown files with clear boundaries). The lack of pre-mortem analysis in Forge is a gap. The lack of human approval in Forge is deliberate (autonomous operation) but could miss important course corrections.

---

### 4. Implementer

| Aspect | Forge (`implementer`) | Gem Team (`gem-implementer`) |
|---|---|---|
| **Scope** | Exactly one task; reads task file, `feature.md`, `design.md` | Exactly one task; reads `plan.yaml` for full task context |
| **Test responsibility** | Writes tests as specified in task; does NOT run them | Full TDD cycle: write failing tests → write minimal code → verify pass |
| **Build responsibility** | Does NOT build | Runs `get_errors` after every edit; runs verification commands |
| **Verification** | None — deferred entirely to verifier agent | Inline verification via `task_block.verification` commands |
| **Refactoring** | Not specified | Optional TDD Refactor step for clarity and DRY |
| **Self-review** | Not specified | Reflection step for medium/high priority tasks (security, performance, naming) |
| **Code style** | No specific guidance | YAGNI/KISS/DRY, functional programming preference, lint compliance |
| **Restricted inputs** | Must NOT read `plan.md` | Reads `plan.yaml` for task context |
| **Security** | Not specified | Never hardcode secrets/PII; OWASP review; fix vulnerabilities before handoff |

**Analysis:**
This is the largest functional difference between the two frameworks. Gem Team's implementer follows strict TDD discipline with inline verification — it confirms tests fail, writes code, and confirms tests pass before returning. Forge's implementer is a pure code writer that defers all execution to the verifier. Gem Team's approach catches issues earlier (at implementation time); Forge's approach provides a cleaner separation of concerns but may result in cascading failures discovered later during verification.

---

### 5. Verifier (Forge) vs. Inline Verification (Gem Team)

| Aspect | Forge (`verifier`) | Gem Team (no dedicated verifier) |
|---|---|---|
| **Existence** | Dedicated agent | No separate verifier — verification is inline per-task |
| **Build** | Builds entire solution | Each implementer runs `get_errors` after edits |
| **Test execution** | Runs full test suite | Each implementer runs task-specific verification commands |
| **Per-task verification** | Cross-references task acceptance criteria against results | Each agent self-verifies against acceptance criteria |
| **Targeted re-verification** | Compares against previous `verifier.md`; focuses on previously failing areas | Replan/re-implement cycle through orchestrator |
| **Report** | Full markdown report with per-task status, build results, test results, actionable items | JSON handoff with status/summary |

**Analysis:**
Forge's dedicated verifier is a clear architectural advantage for complex projects — it provides a single, authoritative verification step with a comprehensive report that drives error recovery. Gem Team distributes verification across implementers, which is faster for simple tasks but can miss integration-level issues that only surface when the full suite runs together.

---

### 6. Reviewer

| Aspect | Forge (`reviewer`) | Gem Team (`gem-reviewer`) |
|---|---|---|
| **Scope** | Full git diff review — maintainability, readability, naming, test quality, architecture | Security-first review — OWASP, secrets, PII, compliance |
| **Review depth** | Single depth — comprehensive peer review | Tiered: Full → Standard → Lightweight based on task priority |
| **Security focus** | General code review (security not explicitly prioritized) | OWASP Top 10, secrets/PII detection, compliance verification |
| **Code modification** | Produces `review.md` with suggestions | Read-only; JSON handoff with review status |
| **Blocking behavior** | Returns `ERROR:` for blocking concerns; orchestrator loops back to planning | Returns `failed` for critical issues; orchestrator routes to planner |
| **Instruction updates** | Updates `.github/instructions` based on codebase changes | Not specified |

**Analysis:**
The two reviewers serve fundamentally different purposes. Forge's reviewer is a peer code reviewer focused on quality and maintainability. Gem Team's reviewer is a security gatekeeper focused on vulnerabilities and compliance. Both are valuable; they emphasize different risk vectors. Forge is missing explicit security scanning. Gem Team is missing a holistic code quality review.

---

### 7. Agents Unique to Forge

#### Spec Agent (`spec`)
Produces a formal specification (`feature.md`) with functional requirements, non-functional requirements, constraints, acceptance criteria, edge cases, user stories, and test scenarios. **Gem Team has no equivalent** — the planner synthesizes research directly into a plan without an explicit specification stage.

**Significance:** The spec stage provides a validated requirements document that both the designer and planner can reference. Without it, design and planning decisions are grounded only in raw research output.

#### Designer (`designer`)
Creates a technical design document (`design.md`) covering architecture, data models, APIs, tradeoffs, migration notes, and testing strategy. **Gem Team has no equivalent** — implementation specification is embedded in the planner's `plan.yaml`.

**Significance:** A dedicated design stage produces deeper architectural analysis and explicit tradeoff documentation. Gem Team's approach embeds this in planning, which can lead to shallower design coverage.

#### Critical Thinker (`critical-thinker`)
Reviews the design by challenging assumptions, probing for gaps, and playing devil's advocate. Forces the designer to reconsider risky decisions before implementation. **Gem Team has no equivalent.**

**Significance:** This is Forge's most distinctive agent. By inserting an adversarial review step between design and planning, Forge catches architectural mistakes before any code is written. This is conceptually similar to Gem Team's pre-mortem analysis in the planner, but operates at the design level rather than the task level — addressing higher-order risks.

---

### 8. Agents Unique to Gem Team

#### Chrome Tester (`gem-chrome-tester`)
Automates browser testing and UI/UX validation via Chrome DevTools. Captures screenshots, validates accessibility (WCAG), runs network/console checks, and executes a validation matrix per task. **Forge has no equivalent.**

**Significance:** For web-facing projects, automated browser testing provides a validation layer that unit and integration tests cannot cover. Forge relies entirely on the verifier's test suite for validation.

#### DevOps (`gem-devops`)
Manages containers (Docker), CI/CD pipelines, Kubernetes orchestration, cloud infrastructure, and deployment automation with health checks and approval gates. **Forge has no equivalent.**

**Significance:** Gem Team can automate the full delivery pipeline from code to deployment. Forge stops at verification and review — deployment is out of scope.

#### Documentation Writer (`gem-documentation-writer`)
Generates technical documentation, API specs (OpenAPI/Swagger), architectural diagrams (Mermaid/PlantUML), and maintains code-documentation parity. **Forge has no equivalent.**

**Significance:** Forge produces workflow artifacts (analysis, spec, design, plan, review) but not end-user or API documentation. Gem Team's doc writer ensures documentation stays synchronized with code changes.

---

## Workflow Comparison

### Forge Pipeline

```
Setup → Research (parallel ×3) → Synthesize → Spec → Design → Critical Review
→ Plan → Implement (parallel waves) → Verify → Review
```

- **Fixed, deterministic flow** — same stages in the same order every time
- **No human pauses** — runs to completion autonomously
- **Rich artifact trail** — 10+ documents produced per feature
- **Error recovery** — retry loops at verification and review stages

### Gem Team Pipeline

```
Research (parallel by focus) → Findings Review (PAUSE) → Plan (with pre-mortem)
→ Plan Approval (PAUSE) → Delegate (parallel ×4, DAG-driven) → Execute + Verify
→ Batch Confirm (PAUSE) → Loop → Delivery Summary
```

- **Dynamic, DAG-driven flow** — task order determined by dependency graph
- **3 mandatory human pauses** — findings, plan approval, batch confirmation
- **YAML state management** — `plan.yaml` tracks all decisions and statuses
- **Per-task agent routing** — different agents for different task types

---

## Strengths and Weaknesses

### Forge Strengths
1. **Dedicated specification and design stages** — produces deeper analysis before any code is written
2. **Critical thinking gate** — adversarial design review catches issues early
3. **Dedicated verifier agent** — single authoritative verification point with comprehensive reporting
4. **Strict agent isolation** — clear I/O boundaries prevent scope creep
5. **Full autonomy** — runs end-to-end without human intervention
6. **Rich artifact trail** — every stage produces a human-readable document
7. **Synthesis step** — dedicated merging of parallel research prevents duplication

### Forge Weaknesses
1. **No pre-mortem analysis** — risks are not formally assessed before execution
2. **No human approval gates** — risky for high-stakes changes
3. **No browser/UI testing** — web-facing features lack visual validation
4. **No DevOps/deployment agent** — pipeline stops at code review, not delivery
5. **No documentation generation** — no agent produces end-user or API docs
6. **No cross-agent memory** — context sharing relies entirely on file artifacts
7. **No task-level verification** — implementer writes code blind; all feedback is deferred to verifier
8. **No TDD enforcement** — implementers write tests but don't run them
9. **No concurrency cap** — unbounded parallelism may overwhelm system resources
10. **Fixed pipeline** — cannot adapt workflow order for non-standard projects

### Gem Team Strengths
1. **Pre-mortem analysis** — proactive risk identification before execution
2. **TDD enforcement** — implementers verify their own work inline
3. **Security-first review** — OWASP scanning, secrets detection, compliance
4. **Broader agent roster** — browser testing, DevOps, documentation
5. **Human approval gates** — safety valve for high-risk decisions
6. **Cross-agent memory** — persistent knowledge with citation verification
7. **YAML state management** — recovery from interruptions, audit trail
8. **Task size limits** — prevents scope creep in individual tasks
9. **Per-task agent routing** — right agent for the right job (e.g., devops tasks go to devops agent)
10. **Concurrency cap** — prevents resource exhaustion

### Gem Team Weaknesses
1. **No explicit specification stage** — requirements are implicit in research + plan
2. **No dedicated design stage** — architecture analysis is embedded in planning
3. **No adversarial design review** — no critical-thinking challenge before implementation
4. **No synthesis step** — research findings go directly to planner without explicit merging
5. **No dedicated verifier** — per-task inline verification can miss integration issues
6. **Mandatory pauses** — slower execution due to 3 human approval gates
7. **Higher complexity** — more agents, YAML schemas, and routing logic to maintain
8. **Fixed pipeline hardcoded to specific tools** — references `dotnet`, specific build commands, etc. (Forge does this too in the verifier)

---

## Summary Matrix

| Capability | Forge | Gem Team |
|---|---|---|
| Parallel research | ✅ (3 concurrent + synthesis) | ✅ (N concurrent, no synthesis) |
| Formal specification | ✅ | ❌ |
| Technical design doc | ✅ | ❌ (embedded in plan) |
| Adversarial design review | ✅ | ❌ |
| Pre-mortem risk analysis | ❌ | ✅ |
| Dependency-aware planning | ✅ (waves) | ✅ (DAG) |
| Parallel implementation | ✅ (wave-based) | ✅ (DAG, max 4) |
| TDD enforcement | ❌ (write only) | ✅ (red-green-refactor) |
| Dedicated verification | ✅ | ❌ (inline per-task) |
| Security scanning | ❌ | ✅ (OWASP, secrets, PII) |
| Browser testing | ❌ | ✅ (Chrome DevTools) |
| DevOps / CI/CD | ❌ | ✅ |
| Documentation generation | ❌ | ✅ |
| Cross-agent memory | ❌ | ✅ |
| Human approval gates | ❌ | ✅ (3 pauses) |
| Autonomous operation | ✅ | ❌ (requires approvals) |
| Error recovery loops | ✅ (max 3) | ✅ (replan/revision) |
| Artifact audit trail | ✅ (markdown files) | ✅ (plan.yaml) |
| Task size guardrails | ❌ | ✅ |
| Concurrency limits | ❌ | ✅ (max 4) |
