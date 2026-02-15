# Optimizing Forge with Gem Team Strengths

This document identifies specific strengths from [Gem Team](https://github.com/mubaidr/gem-team) that are absent in Forge, assesses their value, and provides concrete implementation guidance for incorporating them into the Forge orchestrator workflow.

---

## Priority Legend

| Priority | Meaning |
|---|---|
| **P0 — Critical** | Directly impacts output quality of every run; implement first |
| **P1 — High** | Significant improvement for most projects; implement second |
| **P2 — Medium** | Valuable for specific project types; implement when needed |
| **P3 — Low** | Nice-to-have; defer unless building out the full platform |

---

## 1. Pre-Mortem Analysis in the Planner (P0)

### What Gem Team Does
The `gem-planner` runs a pre-mortem analysis before finalizing the plan. For each high/medium priority task, it identifies potential failure scenarios with likelihood, impact, and mitigation strategies. This is embedded in the `plan.yaml` under `pre_mortem` and per-task `failure_modes`.

### Why Forge Needs This
Forge currently moves from planning directly to implementation with no structured risk assessment. The critical-thinker reviews the design, but no one reviews the plan for execution risks — misestimated dependencies, tasks that might conflict at the integration level, or risky technology choices in specific tasks.

### How to Implement

**Modify `planner.agent.md`:**

Add a "Pre-Mortem Analysis" section to the planner's workflow:

```markdown
## Pre-Mortem Analysis

After creating the task index and execution waves, the planner MUST perform a pre-mortem analysis:

1. For each high or medium priority task, identify at least one failure scenario.
2. For each failure scenario, assess:
   - **Likelihood:** low | medium | high
   - **Impact:** low | medium | high | critical
   - **Mitigation:** concrete action to reduce risk
3. Include an `overall_risk_level` assessment for the entire plan.
4. Document assumptions that, if wrong, would invalidate the plan.

### Pre-Mortem Format in plan.md

```markdown
## Pre-Mortem Analysis

Overall Risk Level: medium

### Assumptions
- The existing auth middleware supports token refresh (if not, task 03 will need redesign)
- The database schema allows nullable columns for backwards compatibility

### Task Risk Assessment
| Task | Scenario | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| 01-create-api | Rate limiting middleware conflicts with existing CORS setup | medium | high | Test middleware ordering in isolation first |
| 03-update-auth | Token refresh endpoint may not exist in current implementation | low | critical | Research agent should verify in architecture analysis |
```
```

**Modify `orchestrator.agent.md`:**

No orchestrator changes needed — the planner produces the pre-mortem as part of `plan.md`. The orchestrator already reads `plan.md` to extract execution waves.

---

## 2. TDD Enforcement in the Implementer (P0)

### What Gem Team Does
The `gem-implementer` follows a strict Red-Green-Refactor TDD cycle:
1. Write failing tests FIRST
2. Confirm tests FAIL
3. Write MINIMAL code to pass
4. Run `get_errors` (compile/lint) after every edit
5. Run verification commands
6. Optional refactor step

### Why Forge Needs This
Forge's implementer writes code and tests but never executes them. All validation is deferred to the verifier. This means:
- Implementers can write tests that don't compile
- Code can have syntax errors that aren't caught until verification
- Multiple tasks in a wave may all fail for the same reason, wasting the verifier's first pass
- The feedback loop is slow — write everything, then discover it doesn't build

### How to Implement

**Modify `implementer.agent.md`:**

Replace the current workflow with a TDD-aware workflow:

```markdown
## Workflow

- Read and understand the task specification
- Write or update unit tests FIRST as specified in the task's test requirements
- Confirm tests fail (run relevant test commands)
- Implement ONLY what the task specifies — write MINIMAL code to pass the tests
- Run `get_errors` after every file edit to catch compile/lint issues immediately
- Confirm tests pass
- (Optional) Refactor for clarity and DRY while keeping tests green
- Update task file:
  - Check acceptance criteria (code-level only)
  - Mark implementation completed

## Rules

- One task only
- No unrelated changes
- No inferred scope
- Focus on writing correct, minimal code that passes the tests
- Run `get_errors` after EVERY edit — do not batch error checking
- If tests fail after implementation, debug and fix before returning
```

**Modify `orchestrator.agent.md`:**

Update the verifier's role description to note that implementers now self-verify at the unit level:

```markdown
- **Implementers perform unit-level TDD** (write tests, write code, confirm pass).
- **The verifier performs integration-level verification** — full build, full test suite, cross-task acceptance criteria.
```

**Modify `verifier.agent.md`:**

Update the role description:

```markdown
## Role

The verifier performs **integration-level verification** after implementation waves complete. While implementers verify their individual tasks at the unit level via TDD, the verifier runs the full build, the complete test suite, and cross-references all task acceptance criteria to catch integration issues.
```

---

## 3. Task Size Limits in the Planner (P1)

### What Gem Team Does
The `gem-planner` enforces hard limits on task scope:
- `max_files: 3` — no task touches more than 3 files
- `max_dependencies: 2` — no task depends on more than 2 others
- `max_lines_to_change: 500` — no task changes more than 500 lines
- `max_estimated_effort: medium` — large tasks must be broken down

### Why Forge Needs This
Without guardrails, the planner can create tasks that are too large for a single implementer invocation. Large tasks increase the risk of failure, make verification harder, and reduce parallelism opportunities.

### How to Implement

**Add to `planner.agent.md`:**

```markdown
## Task Size Limits

Each task MUST stay within these bounds:

| Limit | Maximum |
|---|---|
| Files touched | 3 |
| Dependencies | 2 other tasks |
| Lines changed | 500 |
| Estimated effort | medium |

If a task exceeds any limit, break it into smaller subtasks. Prefer more small tasks over fewer large tasks — this maximizes parallelism and minimizes blast radius on failure.
```

---

## 4. Concurrency Cap (P1)

### What Gem Team Does
The orchestrator enforces a maximum of 4 concurrent agent invocations to prevent resource exhaustion and maintain system stability.

### Why Forge Needs This
Forge dispatches all tasks in a wave concurrently with no limit. For large features with waves containing 8+ tasks, this could overwhelm the system.

### How to Implement

**Modify `orchestrator.agent.md`:**

Add to Global Rules:

```markdown
- **Maximum 4 concurrent subagent invocations.** If a wave contains more than 4 tasks, split it into sub-waves of at most 4 tasks each, dispatching the next sub-wave only after the current one completes.
```

---

## 5. Security-Focused Review (P1)

### What Gem Team Does
The `gem-reviewer` acts as a security gatekeeper with:
- OWASP Top 10 scanning
- Secrets/PII detection via regex scanning
- Compliance verification
- Tiered review depth (Full → Standard → Lightweight) based on task priority

### Why Forge Needs This
Forge's reviewer performs a general peer code review but does not explicitly scan for security vulnerabilities, hardcoded secrets, or compliance issues. Security issues in generated code are a real risk.

### How to Implement

**Modify `reviewer.agent.md`:**

Add a security review section:

```markdown
## Security Review

In addition to code quality review, perform the following security checks:

### For all reviews:
- Scan for hardcoded secrets, API keys, tokens, and passwords
- Check for PII exposure in logs, error messages, and responses
- Verify input validation on all user-facing inputs

### For security-sensitive changes (auth, data, payments, admin):
- OWASP Top 10 scan: injection, broken auth, sensitive data exposure, XXE, broken access control, misconfig, XSS, insecure deserialization, known vulnerabilities, insufficient logging
- Verify authentication and authorization checks on new endpoints
- Check for SQL injection, XSS, and CSRF vulnerabilities

### Review Depth
| Task Priority/Type | Review Depth |
|---|---|
| High priority, security-sensitive, auth, PII, production | Full security scan |
| Medium priority, feature additions | Standard (secrets + basic OWASP) |
| Low priority, bug fixes, minor refactors | Lightweight (secrets scan only) |
```

---

## 6. Per-Task Agent Routing in the Planner (P2)

### What Gem Team Does
Each task in `plan.yaml` has an `agent` field that specifies which agent should execute it — `gem-implementer`, `gem-chrome-tester`, `gem-devops`, `gem-reviewer`, or `gem-documentation-writer`. The orchestrator reads this field to dispatch tasks to the right agent.

### Why Forge Needs This
Forge routes all implementation tasks to the `implementer` agent. If Forge eventually adds specialized agents (browser testing, documentation, etc.), the planner should support routing tasks to specific agents.

### How to Implement (Future-Proofing)

**Modify `planner.agent.md`:**

Add an optional `agent` field to task files:

```markdown
## Task File Requirements

Each task file must include:

- Task goal
- **agent**: (optional) which agent should execute this task. Defaults to `implementer`. Other valid values: any agent defined in the workspace.
- **depends_on**: list of task IDs this task depends on, or `none`
- In-scope / out-of-scope
- ...
```

**Modify `orchestrator.agent.md`:**

Update Step 5.2:

```markdown
2. **Dispatch parallel agents:**
   - For each task in the wave, read the `agent` field from the task file (default: `implementer`).
   - Invoke the specified agent using `runSubagent`.
```

---

## 7. Cross-Agent Memory (P2)

### What Gem Team Does
Agents share knowledge through a memory system:
- Researcher reads memories for project context before exploring
- Planner stores architectural decisions and design patterns
- Orchestrator stores project-level decisions and product vision
- Citations (file:line) are verified before using stored memories

### Why Forge Needs This
Forge relies entirely on file artifacts for inter-agent communication. This works well for a single feature implementation but doesn't carry forward to subsequent features. There's no mechanism for the system to "remember" decisions from previous runs.

### How to Implement

This is a more significant change and depends on the memory infrastructure available in the environment (e.g., VS Code memory MCP tools).

**Minimal approach — add a project decisions file:**

Create a convention for a `docs/decisions.md` file that accumulates architectural decisions across features. The reviewer can append to it; the researcher can read from it.

```markdown
## Decision Log Convention

- The `reviewer` agent SHOULD append significant architectural decisions discovered during review to `docs/decisions.md`.
- The `researcher` agent SHOULD read `docs/decisions.md` (if it exists) as part of architecture research to understand prior decisions.
```

---

## 8. Human Approval Gates (P2)

### What Gem Team Does
Three mandatory pause points:
1. **Findings Review** — after research, before planning
2. **Plan Approval** — after plan creation, before execution
3. **Batch Confirmation** — after each implementation wave, before proceeding

### Why Forge Needs This
Forge is fully autonomous, which is faster but riskier for high-stakes projects. There's no way for a developer to course-correct before code is written.

### How to Implement

**Make this optional, not mandatory.** Add a flag to the prompt file:

**Modify `feature-workflow.prompt.md`:**

```markdown
## Optional: Approval Gates

If `{{APPROVAL_MODE}}` is set to `true`:
- Pause after research synthesis (step 1.2) and present `analysis.md` summary for user review.
- Pause after planning (step 4) and present `plan.md` for user approval.
- Continue implementation only after user confirms.

If `{{APPROVAL_MODE}}` is not set or is `false`:
- Run fully autonomously (current behavior).
```

---

## 9. Hybrid Retrieval Strategy in the Researcher (P1)

### What Gem Team Does
The researcher uses a defined hybrid retrieval strategy:
1. `semantic_search` first for conceptual discovery
2. `grep_search` for exact pattern matching
3. Merge and deduplicate results
4. `read_file` for detailed examination

### Why Forge Needs This
Forge's researcher has no prescribed retrieval methodology. Different invocations may use different strategies, leading to inconsistent research quality.

### How to Implement

**Modify `researcher.agent.md`:**

Add to the Focused Research Rules:

```markdown
### Retrieval Strategy

Use a hybrid retrieval approach in this order:

1. **Semantic search first** — use `semantic_search` to discover relevant concepts, patterns, and related code in the focus area.
2. **Exact pattern matching** — use `grep_search` to find specific function names, class names, keywords, and patterns identified in step 1.
3. **Merge and deduplicate** — combine results from both approaches, removing duplicates.
4. **Detailed examination** — use `read_file` for in-depth analysis of the most relevant files identified in steps 1-2.
5. **File verification** — use `file_search` to confirm file existence. Fall back to external search only if local code is insufficient.

This ensures broad conceptual coverage (semantic) combined with precise pattern identification (grep).
```

---

## 10. Self-Reflection Step for Agents (P3)

### What Gem Team Does
Medium and high-priority tasks trigger a self-reflection step in the implementer, reviewer, and chrome-tester:
- Implementer: self-reviews for security, performance, naming
- Reviewer: self-reviews for completeness and bias
- Chrome tester: self-reviews against acceptance criteria and SLAs

### Why Forge Needs This
Forge agents return results without a structured self-check. A brief reflection step could catch obvious mistakes before the result reaches the orchestrator.

### How to Implement

Add a reflection instruction to `implementer.agent.md` and `reviewer.agent.md`:

```markdown
## Self-Reflection (before returning)

Before returning DONE or ERROR, briefly verify:
- Does the output fully address what was asked?
- Are there any obvious omissions or errors?
- Would a senior engineer approve this output?

If reflection identifies issues, fix them before returning.
```

---

## Implementation Roadmap

### Phase 1 — Immediate (P0)
1. Add pre-mortem analysis to `planner.agent.md`
2. Add TDD enforcement to `implementer.agent.md`

### Phase 2 — Next (P1)
3. Add task size limits to `planner.agent.md`
4. Add concurrency cap to `orchestrator.agent.md`
5. Add security review section to `reviewer.agent.md`
6. Add hybrid retrieval strategy to `researcher.agent.md`

### Phase 3 — When Needed (P2)
7. Add per-task agent routing (when adding new specialized agents)
8. Add project decision log convention
9. Add optional human approval gates

### Phase 4 — Polish (P3)
10. Add self-reflection steps to implementer and reviewer

---

## What NOT to Adopt

Some Gem Team features are **not recommended** for Forge:

| Feature | Why Skip It |
|---|---|
| **YAML state file (`plan.yaml`)** | Markdown artifacts are more human-readable and easier to diff/review. YAML adds complexity without clear benefit for Forge's fixed pipeline. |
| **`disable-model-invocation: true`** | This is a Gem Team-specific mechanism. Forge achieves the same outcome through explicit agent instructions ("never write code"). |
| **Mandatory human pauses** | Forge's value proposition is autonomous execution. Make pauses optional (see item 8) but don't make them mandatory. |
| **Chrome Tester / DevOps / Doc Writer agents** | These are specialized agents for specific project types. Add them as separate optional agents if needed, not as part of the core pipeline. |
