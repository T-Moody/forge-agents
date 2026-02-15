# Feature Specification: Comprehensive Forge Agent Improvements

**Summary:** Rewrite all 9 Forge agents, the workflow prompt, and optionally add a documentation-writer agent — incorporating 30+ improvements from Gem Team analysis, bug fixes, and cross-cutting enhancements — to produce the highest-quality, most robust multi-agent orchestration system possible.

---

## Background & Context

### Source Documents

- [initial-request.md](initial-request.md) — Original user request and constraints
- [analysis.md](analysis.md) — Merged analysis from three independent research streams
- [research/architecture.md](research/architecture.md) — Structural patterns identified in Gem Team
- [research/impact.md](research/impact.md) — Per-agent change specifications
- [research/dependencies.md](research/dependencies.md) — Data flow, coupling clusters, implementation ordering
- [docs/comparison-forge-vs-gem-team.md](../../comparison-forge-vs-gem-team.md) — Prior agent-by-agent comparison
- [docs/optimization-from-gem-team.md](../../optimization-from-gem-team.md) — Prior improvement guidance

### Premise

The user has granted **full permission to alter any Forge agent in any way that makes it better**. There are no sacred cows, no architecture constraints. Every improvement identified in the analysis is included unless there is a concrete technical reason not to. All agents are being rewritten from scratch into `NewAgentsAndPrompts/`, so migration cost within the agent files themselves is zero. The only real constraints are preserving the orchestrator's ability to coordinate agents and maintaining the artifact-based communication model that is a genuine Forge strength.

### Forge Architecture (Preserved)

The following core architectural properties are **retained** because they are genuine strengths validated by analysis:

1. Fixed deterministic 8-step pipeline (predictable, debuggable)
2. Markdown artifact chain (human-readable, diffable, consistent)
3. Orchestrator as sole router (no agent-to-agent communication)
4. `DONE:`/`ERROR:` completion contracts (simple, text-based)
5. Dedicated spec, design, and critical-thinker stages (unique Forge value)
6. Dedicated verifier agent (single integration verification point)
7. Research synthesis step (prevents duplication)
8. Full autonomy by default (end-to-end without human intervention)
9. Execution wave model with individual task files
10. Verifier→Planner and Reviewer→Planner feedback loops with bounded retries

---

## Functional Requirements

All improved agent files and prompts are output to `NewAgentsAndPrompts/`. Each subsection specifies exactly what changes, which file(s) are affected, and what the expected behavior is after implementation.

### FR-1: Orchestrator (`orchestrator.agent.md`)

#### FR-1.1: Update TDD Global Rule [P0 — Cluster A]

- **What:** Replace the global rule "Implementers never build or run tests" with "Implementers perform unit-level TDD (write and run tests for their task). The verifier performs integration-level verification across all tasks."
- **File:** `orchestrator.agent.md`
- **Expected behavior:** Orchestrator no longer prohibits implementers from running tests. Verifier's role is clearly scoped to integration verification.
- **Coupled with:** FR-7.1, FR-8.1 — all three must change simultaneously.

#### FR-1.2: Concurrency Cap [P1]

- **What:** Add rule: "Maximum 4 concurrent subagent invocations. If a wave contains more than 4 tasks, partition into sub-waves of ≤4 tasks. Dispatch each sub-wave sequentially, waiting for all tasks in a sub-wave to complete before dispatching the next."
- **File:** `orchestrator.agent.md` (global rules + Step 5.2)
- **Expected behavior:** No wave dispatches more than 4 implementers concurrently. Waves with 5+ tasks are automatically split.

#### FR-1.3: Concurrency Cap Reinforcement in Prompt [P1]

- **What:** Add concurrency cap rule to the workflow prompt to reinforce the orchestrator rule.
- **File:** `feature-workflow.prompt.md`
- **Expected behavior:** Both orchestrator and prompt agree on the max-4 cap.

#### FR-1.4: Per-Task Agent Routing [P2 — Cluster B]

- **What:** In Step 5.2, before dispatching each task, read the `agent` field from the task file. If present, dispatch to the specified agent instead of defaulting to `implementer`. If absent or unrecognized, default to `implementer`.
- **File:** `orchestrator.agent.md`
- **Expected behavior:** Tasks with `agent: documentation-writer` are dispatched to the documentation-writer agent. Tasks without an `agent` field go to the implementer as before.
- **Coupled with:** FR-6.7

#### FR-1.5: Decision Log in Artifact List [P2]

- **What:** Add `decisions.md` to the orchestrator's documentation structure as an optional cross-feature decision log. Note that the reviewer writes to it and the researcher reads from it if it exists.
- **File:** `orchestrator.agent.md`
- **Expected behavior:** Orchestrator is aware of `decisions.md` as a valid artifact.

#### FR-1.6: Optional Human Approval Gates [P2 — Cluster C]

- **What:** Add conditional pause logic: if `{{APPROVAL_MODE}}` is set to `true` in the prompt, pause after research synthesis (Step 2) and after planning (Step 4) for human approval before continuing. If `{{APPROVAL_MODE}}` is `false` or unset, run fully autonomously (current default behavior).
- **Files:** `orchestrator.agent.md`, `feature-workflow.prompt.md`
- **Expected behavior:** Default behavior is unchanged (fully autonomous). When APPROVAL_MODE is explicitly enabled, the orchestrator pauses at two strategic points.
- **Coupled with:** FR-1.6 prompt changes

#### FR-1.7: Anti-Drift Anchor [P2]

- **What:** Add a final instruction block at the end of the orchestrator instructions repeating core behavioral constraints: "You are the orchestrator. You coordinate agents. You never write code, tests, or documentation directly. You never skip pipeline steps. Stay as orchestrator."
- **File:** `orchestrator.agent.md`
- **Expected behavior:** Reduces risk of mode-switching in long orchestration sessions.

#### FR-1.8: Context-Efficient Reading Guidance [P3]

- **What:** If the orchestrator ever reads files (e.g., task files for routing), prefer targeted line-range reads; limit to ~200 lines per `read_file` call.
- **File:** `orchestrator.agent.md`
- **Expected behavior:** Reduces context window waste.

#### FR-1.9: `detailed thinking on` Directive [P3]

- **What:** Add chain-of-thought activation directive near the start of the agent's instructions.
- **File:** `orchestrator.agent.md`
- **Expected behavior:** Activates extended reasoning (model-dependent).

---

### FR-2: Researcher (`researcher.agent.md`)

#### FR-2.1: Hybrid Retrieval Strategy [P1]

- **What:** Add a "Retrieval Strategy" subsection to Focused Research Rules prescribing a specific methodology:
  1. `semantic_search` for conceptual discovery (find relevant code by meaning)
  2. `grep_search` for exact pattern matching (find specific identifiers, strings, configurations)
  3. Merge and deduplicate results from both approaches
  4. `read_file` for detailed examination of identified files (targeted line ranges)
  5. `file_search` for existence verification of expected files/patterns
- **File:** `researcher.agent.md`
- **Expected behavior:** Researcher follows a consistent, thorough retrieval methodology rather than ad-hoc searching. Research output is more complete and consistent.

#### FR-2.2: Confidence Metadata [P2]

- **What:** Add optional metadata fields to the partial analysis file format:
  - `confidence_level`: high / medium / low
  - `coverage_estimate`: qualitative description of how much of the relevant codebase was examined
  - `gaps`: list of areas not covered, with impact assessment
- **File:** `researcher.agent.md`
- **Expected behavior:** Downstream agents (spec, planner) can gauge research completeness and identify areas needing additional investigation.

#### FR-2.3: Decision Log Read Convention [P2]

- **What:** During architecture-focused research, if `decisions.md` exists in the documentation structure, read it and incorporate prior architectural decisions into the analysis.
- **File:** `researcher.agent.md`
- **Expected behavior:** Research is informed by prior decisions, reducing repeated investigation of already-decided questions.

#### FR-2.4: Context-Efficient Reading Guidance [P3]

- **What:** Add: "Prefer semantic search, file outlines, and targeted line-range reads; limit to ~200 lines per `read_file` call. Use `semantic_search` and `grep_search` for discovery before reading full files."
- **File:** `researcher.agent.md`
- **Expected behavior:** Reduces context window waste during research.

#### FR-2.5: Anti-Drift Anchor [P2]

- **What:** Add final instruction block: "You are the researcher. You investigate and document findings. You never modify source code, tests, or project files. You never make design decisions. Stay as researcher."
- **File:** `researcher.agent.md`

#### FR-2.6: `detailed thinking on` Directive [P3]

- **What:** Add chain-of-thought activation directive near the start of the agent's instructions.
- **File:** `researcher.agent.md`

---

### FR-3: Spec (`spec.agent.md`)

#### FR-3.1: Structured Edge Case Format [P3]

- **What:** Add guidance that each edge case should include:
  - **Input/Condition**: What triggers the edge case
  - **Expected Behavior**: What should happen
  - **Severity if Missed**: Impact of not handling this case (critical/high/medium/low)
- **File:** `spec.agent.md`
- **Expected behavior:** Edge cases in `feature.md` are structured and assessable, not just prose descriptions.

#### FR-3.2: Acceptance Criteria Verification Step [P3]

- **What:** Add a verification step before returning: "Verify that all acceptance criteria are testable — each criterion should have a clear pass/fail definition that can be verified by the verifier agent."
- **File:** `spec.agent.md`
- **Expected behavior:** Spec agent self-checks that its acceptance criteria are actionable.

#### FR-3.3: Anti-Drift Anchor [P2]

- **What:** Add final instruction block: "You are the spec agent. You write formal requirements specifications. You never write code, designs, or plans. Stay as spec."
- **File:** `spec.agent.md`

#### FR-3.4: `detailed thinking on` Directive [P3]

- **What:** Add chain-of-thought activation directive.
- **File:** `spec.agent.md`

---

### FR-4: Designer (`designer.agent.md`)

#### FR-4.1: Security Considerations Section [P2]

- **What:** Add a "Security Considerations" subsection to the design.md output format covering:
  - Authentication/authorization patterns
  - Data protection approach
  - Threat model (high-level)
  - Input validation strategy
- **File:** `designer.agent.md`
- **Expected behavior:** `design.md` includes a security section. For features with no security implications, the section may state "No security considerations identified" with justification.

#### FR-4.2: Failure & Recovery Section [P2]

- **What:** Add a "Failure & Recovery" subsection to the design.md output format covering:
  - Expected failure modes
  - Retry/fallback strategies
  - Graceful degradation approach
- **File:** `designer.agent.md`
- **Expected behavior:** `design.md` addresses how the designed system handles failures. Complements FR-6.1 (planner pre-mortem) by addressing failure at the design level rather than the task level.

#### FR-4.3: Self-Verification Step [P3]

- **What:** Before returning, verify that the design addresses all functional and non-functional requirements from `feature.md`, and that every acceptance criterion has a clear implementation path in the design.
- **File:** `designer.agent.md`
- **Expected behavior:** Designer self-checks completeness against the spec before returning.

#### FR-4.4: Anti-Drift Anchor [P2]

- **What:** Add final instruction block: "You are the designer. You create technical design documents. You never write code, tests, or plans. You never implement anything. Stay as designer."
- **File:** `designer.agent.md`

#### FR-4.5: `detailed thinking on` Directive [P3]

- **What:** Add chain-of-thought activation directive.
- **File:** `designer.agent.md`

---

### FR-5: Critical Thinker (`critical-thinker.agent.md`)

#### FR-5.1: Completion Contract [P0 — Bug Fix]

- **What:** Add a `## Completion Contract` section specifying that the agent must end its response with either `DONE: <summary of critical review>` or `ERROR: <reason>`.
- **File:** `critical-thinker.agent.md`
- **Expected behavior:** Critical thinker produces a completion line the orchestrator can parse, matching the contract every other agent has.

#### FR-5.2: Output File Specification [P0 — Bug Fix]

- **What:** Add explicit output specification: the critical thinker writes its review to `design_critical_review.md` in the feature documentation directory. This file name must match what the orchestrator references in Step 3b.
- **File:** `critical-thinker.agent.md`
- **Expected behavior:** No ambiguity about where the critical review is written.

#### FR-5.3: Structured Risk Categories [P2]

- **What:** Add specific risk categories the critical thinker should probe:
  - Security vulnerabilities
  - Scalability bottlenecks
  - Maintainability concerns
  - Backwards compatibility risks
  - Edge cases not covered by the design
  - Performance implications
- Add instruction: "Ground each risk in specific technical details from `design.md`. Avoid generic risks that could apply to any project."
- **File:** `critical-thinker.agent.md`
- **Expected behavior:** Critical review is structured around concrete risk categories and grounded in the actual design, not abstract concerns.

#### FR-5.4: Context-Efficient Reading Guidance [P3]

- **What:** Add reading guidance (same as FR-2.4).
- **File:** `critical-thinker.agent.md`

#### FR-5.5: Anti-Drift Anchor [P2]

- **What:** Add final instruction block: "You are the critical thinker. You perform adversarial design reviews. You never write code, designs, plans, or specifications. You identify risks and weaknesses. Stay as critical thinker."
- **File:** `critical-thinker.agent.md`

#### FR-5.6: `detailed thinking on` Directive [P3]

- **What:** Add chain-of-thought activation directive.
- **File:** `critical-thinker.agent.md`

---

### FR-6: Planner (`planner.agent.md`)

#### FR-6.1: Pre-Mortem Analysis [P0]

- **What:** After creating the task index and execution waves, the planner performs a pre-mortem analysis and appends it as a new section in `plan.md`:
  - Per-task failure scenarios with likelihood (high/medium/low), impact (high/medium/low), and mitigation strategy
  - Overall risk level for the plan
  - Key assumptions that, if wrong, would invalidate the plan
- **File:** `planner.agent.md`
- **Expected behavior:** `plan.md` includes a "Pre-Mortem Analysis" section. Implementers and the verifier can reference it to anticipate and handle failures proactively.

#### FR-6.2: Task Size Limits [P1]

- **What:** Add hard limits for individual tasks. Any task exceeding any limit must be broken into subtasks:
  - Maximum 3 files touched per task
  - Maximum 2 task dependencies
  - Maximum 500 lines changed
  - Maximum "medium" effort rating
- **File:** `planner.agent.md`
- **Expected behavior:** No task in `plan.md` exceeds these limits. Tasks that would exceed them are automatically decomposed into smaller subtasks during planning.

#### FR-6.3: Mode Awareness [P1]

- **What:** Add explicit mode detection at the top of the planner's workflow:
  - **Initial**: No `plan.md` exists — create a full plan from scratch
  - **Replan**: `verifier.md` exists with failures — create a remediation plan addressing specific failures
  - **Extension**: Existing `plan.md` with new objectives — extend the plan without duplicating already-completed work
- **File:** `planner.agent.md`
- **Expected behavior:** Planner explicitly identifies which mode it is operating in and adjusts its behavior accordingly. Currently handles replan implicitly but doesn't distinguish modes.

#### FR-6.4: Per-Task Agent Field [P2 — Cluster B]

- **What:** Add optional `agent` field to task file format. Default value: `implementer`. Other valid values: `documentation-writer` (if FR-11.1 is implemented), or any future agent name.
- **File:** `planner.agent.md`
- **Expected behavior:** Planner can assign tasks to agents other than the implementer. The orchestrator reads this field (FR-1.4) to route tasks.
- **Coupled with:** FR-1.4

#### FR-6.5: Implementation Specification [P2]

- **What:** Add optional "Implementation Specification" section to `plan.md` containing:
  - Code structure overview (new packages/modules/files to create)
  - Affected areas (existing code that will be modified)
  - Integration points (where new code connects to existing code)
- This section should **reference** `design.md` rather than duplicating its content. It bridges the gap between high-level design and individual task instructions.
- **File:** `planner.agent.md`
- **Expected behavior:** Implementers have clearer architectural context for their tasks.

#### FR-6.6: YAGNI/KISS/DRY Guidance [P3]

- **What:** Add planning principles: "Prefer simpler decompositions. Avoid creating unnecessary tasks. Each task should deliver tangible, testable value. Do not plan work that isn't required by the specification."
- **File:** `planner.agent.md`
- **Expected behavior:** Plans are leaner and more focused.

#### FR-6.7: Context-Efficient Reading Guidance [P3]

- **What:** Add reading guidance (same as FR-2.4).
- **File:** `planner.agent.md`

#### FR-6.8: Anti-Drift Anchor [P2]

- **What:** Add final instruction block: "You are the planner. You decompose work into tasks and execution waves. You never write code, tests, or documentation. You never implement tasks. Stay as planner."
- **File:** `planner.agent.md`

#### FR-6.9: `detailed thinking on` Directive [P3]

- **What:** Add chain-of-thought activation directive.
- **File:** `planner.agent.md`

---

### FR-7: Implementer (`implementer.agent.md`)

**Most significantly changed agent.**

#### FR-7.1: TDD Enforcement [P0 — Cluster A]

- **What:** Replace the implementer's existing workflow with a Test-Driven Development cycle:
  1. Read and understand the assigned task file
  2. Write failing test(s) for the task's requirements
  3. Run tests — confirm they fail (validates the test is meaningful)
  4. Write minimal production code to make tests pass
  5. Run `get_errors` after every file edit
  6. Run tests — confirm they pass
  7. Refactor if needed (tests must still pass after refactoring)
  8. Update task file with completion status
- Remove the rules "Do NOT build the project" and "Do NOT run tests"
- Update the frontmatter description to reflect the TDD workflow
- Add rule: "Run `get_errors` after every file edit to catch compilation/lint errors immediately"
- **File:** `implementer.agent.md`
- **Expected behavior:** Implementer writes tests first, then code. Every task has associated tests. Build/test errors are caught during implementation, not deferred to verification.
- **Coupled with:** FR-1.1, FR-8.1
- **Fallback:** If the project has no test framework configured or the task is purely configuration/documentation, the implementer should note this in the task file and proceed without TDD, focusing on `get_errors` validation instead.

#### FR-7.2: Security Rules [P1]

- **What:** Add security rules:
  - Never hardcode secrets, API keys, tokens, or passwords — use environment variables or configuration files
  - Never expose PII in logs, error messages, or comments
  - If a security vulnerability is identified during implementation, fix it before returning (or flag it if fixing would exceed task scope)
- **File:** `implementer.agent.md`
- **Expected behavior:** Implementer actively avoids introducing security issues.

#### FR-7.3: Code Quality Principles [P1]

- **What:** Add:
  - YAGNI: Don't implement functionality that isn't required by the task
  - KISS: Prefer the simplest correct solution
  - DRY: Extract duplication only when there are 3+ instances (avoid premature abstraction)
  - Before refactoring or modifying existing code, use `list_code_usages` to check all call sites and ensure no breakage
- **File:** `implementer.agent.md`
- **Expected behavior:** Implementer writes focused, minimal, correct code and checks impact before refactoring.

#### FR-7.4: Self-Reflection Step [P3]

- **What:** Before returning, the implementer verifies:
  - All task requirements are addressed
  - Tests pass and cover the task's acceptance criteria
  - No obvious omissions or incomplete implementations
  - "Would a senior engineer approve this code?"
  - Fix any issues found during self-reflection before returning
- **File:** `implementer.agent.md`
- **Expected behavior:** Implementer self-checks quality before returning, catching issues that would otherwise be found by the verifier or reviewer.

#### FR-7.5: Anti-Drift Anchor [P2]

- **What:** Add final instruction block: "You are the implementer. You write code and tests for exactly one task. You never modify other tasks' files. You never skip TDD steps. You never modify plan.md. Stay as implementer."
- **File:** `implementer.agent.md`

#### FR-7.6: `detailed thinking on` Directive [P3]

- **What:** Add chain-of-thought activation directive.
- **File:** `implementer.agent.md`

---

### FR-8: Verifier (`verifier.agent.md`)

#### FR-8.1: Role Redefinition to Integration-Level Verifier [P0 — Cluster A]

- **What:** Redefine the verifier's role from "sole owner of building and testing" to "integration-level verifier." Add note: "Implementers have already verified their individual tasks via unit-level TDD. The verifier focuses on:
  - Full project build (does everything compile/link together?)
  - Full test suite execution (including implementer-written unit tests + any integration/e2e tests)
  - Cross-task interaction verification (do independently-implemented tasks work together?)
  - Acceptance criteria verification against `feature.md`"
- **File:** `verifier.agent.md`
- **Expected behavior:** Verifier understands that unit tests should already pass and focuses on integration-level concerns. If unit tests fail, this indicates an implementer issue to flag.
- **Coupled with:** FR-1.1, FR-7.1

#### FR-8.2: Technology-Agnostic Build/Test Commands [P1]

- **What:** Replace all hardcoded commands (`dotnet build TourneyPal.sln`, `dotnet test tests/TourneyPal.Tests`) with technology-agnostic instructions:
  1. Detect the project's build system by scanning for build configuration files (in priority order): `package.json` (npm/yarn/pnpm), `Makefile`/`CMakeLists.txt`, `*.sln`/`*.csproj` (.NET), `pom.xml`/`build.gradle` (Java), `Cargo.toml` (Rust), `pyproject.toml`/`setup.py` (Python), `go.mod` (Go)
  2. Run the appropriate build command for the detected system
  3. Run the appropriate test command for the detected system
  4. If multiple build systems are detected, prefer the one referenced in project documentation or `README.md`
  5. If no build system is detected, report this as an error
- **File:** `verifier.agent.md`
- **Expected behavior:** Verifier works with any technology stack, not just .NET.

#### FR-8.3: Read-Only Enforcement [P1]

- **What:** Add explicit rule: "The verifier MUST NOT modify source code, test files, configuration files, or any project files. The verifier is strictly read-only with respect to the codebase. The only files the verifier writes are its output artifact (`verifier.md`) and task file status updates."
- **File:** `verifier.agent.md`
- **Expected behavior:** Verifier never accidentally "fixes" code — it only reports findings.

#### FR-8.4: Context-Efficient Reading Guidance [P3]

- **What:** Add reading guidance (same as FR-2.4).
- **File:** `verifier.agent.md`

#### FR-8.5: Anti-Drift Anchor [P2]

- **What:** Add final instruction block: "You are the verifier. You build, test, and verify. You never modify source code or fix bugs. You report findings. Stay as verifier."
- **File:** `verifier.agent.md`

#### FR-8.6: `detailed thinking on` Directive [P3]

- **What:** Add chain-of-thought activation directive.
- **File:** `verifier.agent.md`

---

### FR-9: Reviewer (`reviewer.agent.md`)

#### FR-9.1: Security Review Section [P1]

- **What:** Add a security review component:
  - **For all reviews:** Scan for hardcoded secrets, API keys, tokens, passwords, PII in logs/error messages using `grep_search` with patterns like `password`, `secret`, `api_key`, `token`, `Bearer`, etc.
  - **For security-sensitive changes** (auth, data storage, payments, admin, networking): Apply OWASP Top 10 checklist (injection, broken auth, sensitive data exposure, XXE, broken access control, misconfiguration, XSS, insecure deserialization, known vulnerabilities, insufficient logging)
  - Security issues should be flagged as `severity: critical` in the review output
- **File:** `reviewer.agent.md`
- **Expected behavior:** Every review includes a security scan. Security-sensitive features get deeper OWASP-based review.

#### FR-9.2: Tiered Review Depth [P2]

- **What:** Add tiered review depth determined by examining the diff content:
  - **Full**: Security-sensitive changes (auth, data, payments, admin), core architecture changes, public API changes → review every line, check all criteria
  - **Standard**: Business logic, new features, refactoring → review logic correctness, test coverage, naming, patterns
  - **Lightweight**: Documentation, configuration, dependency updates, formatting → check correctness and consistency only
- The reviewer determines the tier by examining the changed files and their content, not from external metadata.
- **File:** `reviewer.agent.md`
- **Expected behavior:** Review effort is proportional to the risk level of the changes.

#### FR-9.3: Read-Only Enforcement [P1]

- **What:** Add explicit rule: "The reviewer MUST NOT modify source code, test files, or project files. The reviewer is strictly read-only. The only files the reviewer writes are `review.md`, `.github/instructions/*.instructions.md` updates, and optionally `decisions.md`."
- **File:** `reviewer.agent.md`
- **Expected behavior:** Reviewer never accidentally modifies code.

#### FR-9.4: Decision Log Write Convention [P2]

- **What:** When the reviewer identifies significant architectural decisions during review (e.g., "this pattern should be used consistently across the codebase," "this dependency was chosen over X for reason Y"), append them to `decisions.md` with date, context, and rationale.
- **File:** `reviewer.agent.md`
- **Expected behavior:** Architectural decisions are captured for future reference.

#### FR-9.5: Quality Bar Heuristic [P3]

- **What:** Add calibration standard: "Apply the standard: Would a staff engineer approve this code? This means: correct, well-tested, readable, maintainable, secure, and following project conventions."
- **File:** `reviewer.agent.md`
- **Expected behavior:** Reviewer has a clear quality benchmark.

#### FR-9.6: Self-Reflection Step [P3]

- **What:** Before returning, verify: all changed files were reviewed, no files were skipped, review comments are actionable and specific, security scan was completed.
- **File:** `reviewer.agent.md`
- **Expected behavior:** Reviewer self-checks completeness.

#### FR-9.7: Context-Efficient Reading Guidance [P3]

- **What:** Add reading guidance (same as FR-2.4).
- **File:** `reviewer.agent.md`

#### FR-9.8: Anti-Drift Anchor [P2]

- **What:** Add final instruction block: "You are the reviewer. You review code for quality, correctness, and security. You never modify source code. You write review findings only. Stay as reviewer."
- **File:** `reviewer.agent.md`

#### FR-9.9: `detailed thinking on` Directive [P3]

- **What:** Add chain-of-thought activation directive.
- **File:** `reviewer.agent.md`

---

### FR-10: Workflow Prompt (`feature-workflow.prompt.md`)

#### FR-10.1: Concurrency Cap Reinforcement [P1]

- **What:** Add to the prompt: "Maximum 4 concurrent subagent invocations per wave."
- **File:** `feature-workflow.prompt.md`

#### FR-10.2: APPROVAL_MODE Variable [P2 — Cluster C]

- **What:** Add `{{APPROVAL_MODE}}` variable documentation and conditional logic. Default: `false` (fully autonomous). When `true`: orchestrator pauses after research synthesis and after planning.
- **File:** `feature-workflow.prompt.md`
- **Coupled with:** FR-1.6

#### FR-10.3: Per-Task Agent Routing Rule [P2]

- **What:** Add rule: "Tasks may specify an `agent` field. The orchestrator dispatches to the named agent (default: implementer)."
- **File:** `feature-workflow.prompt.md`

---

### FR-11: Documentation Writer (`documentation-writer.agent.md`) — NEW, Optional

#### FR-11.1: Create Documentation Writer Agent [P2]

- **What:** Create a new agent for documentation tasks:
  - **Role:** Generates and maintains project documentation
  - **Capabilities:** API documentation (OpenAPI/Swagger), architectural diagrams (Mermaid), README updates, code-documentation parity verification, documentation coverage matrix
  - **Constraints:** Read-only access to source code (never modifies it); writes only documentation files
  - **Invocation:** Not in the core pipeline — invoked only when the planner assigns documentation tasks via the `agent` field (requires FR-6.4 + FR-1.4)
  - **Completion contract:** `DONE: <summary>` / `ERROR: <reason>` (consistent with all Forge agents)
  - **Anti-drift anchor:** Included
- **File:** `documentation-writer.agent.md` (new file)
- **Depends on:** FR-1.4 (per-task agent routing) and FR-6.4 (agent field in tasks)
- **Expected behavior:** When the planner creates a task with `agent: documentation-writer`, the orchestrator dispatches it to this agent, which produces documentation artifacts.

---

### FR-12: Cross-Cutting — Shared Operating Rules

#### FR-12.1: Error Handling Guidance [P2]

- **What:** Add to all agents that interact with tools (researcher, implementer, verifier, reviewer, planner, critical-thinker):
  - **Transient errors** (network timeout, tool unavailable, rate limit): Retry up to 2 times with brief delay
  - **Persistent errors** (file not found, syntax error in source, permission denied): Include in output and continue; do not retry
  - **Security issues** (secrets in code, vulnerable dependencies): Flag immediately in output with `severity: critical`
  - **Missing context** (referenced file doesn't exist, dependency not installed): Note the gap and proceed with available information; do not block
- **Files:** All applicable agent files
- **Expected behavior:** Agents handle errors consistently rather than failing unpredictably.

---

## Non-Functional Requirements

### NFR-1: Consistency

- All agents must use the same completion contract format: `DONE: <summary>` / `ERROR: <reason>`
- All agents must have anti-drift anchors (FR-\*. Anti-Drift Anchor requirements)
- All agents that read the codebase must include context-efficient reading guidance
- Error handling guidance must be consistent across all agents (FR-12.1)

### NFR-2: Portability

- No agent may contain hardcoded project names, technology stacks, file paths, or solution names
- Build/test commands must be technology-agnostic (detected at runtime)
- Agent definitions must work with any software project, not just .NET or any specific stack

### NFR-3: Quality

- Every agent improvement must be self-consistent — no agent should contain contradictory instructions
- No agent should reference rules, files, or behaviors defined in other agents unless those dependencies are explicitly documented
- Each agent file must be complete and self-contained (aside from the shared operating rules that are duplicated into each file)

### NFR-4: Token Efficiency

- Context-efficient reading guidance reduces unnecessary token consumption
- Anti-drift anchors are concise (3-5 lines maximum per agent)
- Agents should not include verbose commentary or examples unless they directly improve task execution

### NFR-5: Robustness

- TDD fallback must exist for projects without test frameworks (FR-7.1 fallback)
- Technology detection must handle unrecognized build systems gracefully (FR-8.2)
- Per-task agent routing must default to `implementer` when the `agent` field is absent or unrecognized (FR-1.4)
- APPROVAL_MODE must default to autonomous when unset (FR-1.6)

---

## Constraints & Assumptions

### Constraints

1. **Output directory:** All new/improved agent files are written to `NewAgentsAndPrompts/` — never to `.github/`
2. **Do not modify `.github/`:** The active agents in `.github/agents/` and `.github/prompts/` are orchestrating this work and must not be changed
3. **File format:** All agent files use the `chatagent` markdown format with YAML frontmatter
4. **Completion contract:** All agents must use `DONE:`/`ERROR:` text-based completion (not JSON, not three-state)
5. **Orchestrator sovereignty:** The orchestrator remains the sole dispatcher — agents never invoke other agents directly
6. **Markdown artifacts:** All inter-agent communication remains via markdown files (not YAML, not JSON, not shared mutable state)

### Assumptions

1. The `chatagent` format supports standard YAML frontmatter fields (`name`, `description`, `tools`, `applyTo`)
2. The `{{APPROVAL_MODE}}` template variable can be passed through VS Code Copilot's prompt system
3. `list_code_usages` is available as a tool in the target environment (if not, the implementer should use `grep_search` as a fallback)
4. `get_errors` is available as a tool for the implementer's TDD workflow
5. The target environment supports running tests via terminal commands
6. Task size limits (3 files, 500 lines, 2 dependencies) are appropriate defaults — these may need calibration based on project characteristics
7. Concurrency cap of 4 is appropriate — may need adjustment based on environment

---

## Acceptance Criteria

### AC-1: All Files Produced

All of the following files exist in `NewAgentsAndPrompts/`:

- `orchestrator.agent.md`
- `researcher.agent.md`
- `spec.agent.md`
- `designer.agent.md`
- `critical-thinker.agent.md`
- `planner.agent.md`
- `implementer.agent.md`
- `verifier.agent.md`
- `reviewer.agent.md`
- `feature-workflow.prompt.md`
- `documentation-writer.agent.md`

**Pass:** All 11 files exist and are non-empty.
**Fail:** Any file is missing or empty.

### AC-2: Bug Fixes Applied

- `critical-thinker.agent.md` contains a `## Completion Contract` section with `DONE:`/`ERROR:` format
- `critical-thinker.agent.md` specifies `design_critical_review.md` as its output file

**Pass:** Both sections exist and match what the orchestrator expects.
**Fail:** Either section is missing or inconsistent with the orchestrator.

### AC-3: TDD Enforcement (Cluster A Coordination)

- `implementer.agent.md` contains a TDD workflow (write tests → fail → code → pass → refactor)
- `implementer.agent.md` does NOT contain "Do NOT build" or "Do NOT run tests" rules
- `implementer.agent.md` requires `get_errors` after every file edit
- `verifier.agent.md` defines its role as "integration-level verification" (not "sole owner of testing")
- `orchestrator.agent.md` global rules state implementers perform unit-level TDD

**Pass:** All 5 conditions are true.
**Fail:** Any condition is false, or any file contradicts another.

### AC-4: Concurrency Cap

- `orchestrator.agent.md` specifies max 4 concurrent subagent invocations
- `orchestrator.agent.md` includes sub-wave splitting logic for waves >4 tasks
- `feature-workflow.prompt.md` reinforces the max-4 cap

**Pass:** All 3 conditions are true.
**Fail:** Any condition is false or the cap values are inconsistent.

### AC-5: Planner Improvements

- `planner.agent.md` includes pre-mortem analysis output format
- `planner.agent.md` includes task size limits (3 files, 2 dependencies, 500 lines, medium max effort)
- `planner.agent.md` includes mode detection (initial/replan/extension)

**Pass:** All 3 sections exist and are well-specified.
**Fail:** Any section is missing or underspecified.

### AC-6: Security Thread

- `implementer.agent.md` includes security rules (no hardcoded secrets, no PII in logs)
- `reviewer.agent.md` includes security review section (secrets scan + OWASP for sensitive changes)
- `designer.agent.md` includes security considerations section

**Pass:** All 3 agents address security in their domain.
**Fail:** Any agent is missing its security component.

### AC-7: Technology Portability

- `verifier.agent.md` does NOT contain any hardcoded project names, solution files, or technology-specific commands
- `verifier.agent.md` includes technology detection logic

**Pass:** No hardcoded references; detection logic is present.
**Fail:** Any hardcoded project/technology reference remains.

### AC-8: Anti-Drift Anchors

- All 9 agent files (+ documentation-writer) contain a final anti-drift anchor block
- Each anchor correctly identifies the agent's role and core constraints

**Pass:** All 10 agents have anchors.
**Fail:** Any agent is missing an anchor or has an incorrect role identity.

### AC-9: Cross-Cutting Consistency

- All agents have `DONE:`/`ERROR:` completion contracts
- All agents that read the codebase include context-efficient reading guidance
- No agent contains instructions that contradict another agent's instructions

**Pass:** All conditions are true.
**Fail:** Any inconsistency or contradiction exists between agents.

### AC-10: Per-Task Agent Routing (Cluster B)

- `planner.agent.md` defines an optional `agent` field in its task file format
- `orchestrator.agent.md` reads the `agent` field and dispatches accordingly
- `feature-workflow.prompt.md` documents per-task routing

**Pass:** All 3 files are coordinated.
**Fail:** Any file is missing the routing support, or the field names are inconsistent.

### AC-11: Optional Approval Gates (Cluster C)

- `orchestrator.agent.md` includes conditional pause logic for `APPROVAL_MODE`
- `feature-workflow.prompt.md` defines the `{{APPROVAL_MODE}}` variable
- Default behavior (APPROVAL_MODE unset or false) is fully autonomous

**Pass:** Gates work when enabled and are invisible when disabled.
**Fail:** Gates are always active, or the variable is not defined.

### AC-12: Documentation Writer Agent

- `documentation-writer.agent.md` exists with role definition, workflow, completion contract, and anti-drift anchor
- Agent is not part of the core pipeline (invoked only via per-task routing)

**Pass:** File exists and is complete; not referenced in the core pipeline steps.
**Fail:** File is missing, incomplete, or hardwired into the pipeline.

---

## Edge Cases & Error Handling

### EC-1: TDD in Non-Testable Contexts

- **Condition:** Project has no test framework, or the task is purely configuration/documentation
- **Expected behavior:** Implementer notes the absence of a test framework in the task file, skips test-writing steps, and proceeds with code implementation using `get_errors` validation only
- **Must NOT:** Fail or return ERROR solely because TDD cannot be performed

### EC-2: Unrecognized Build System

- **Condition:** Verifier cannot detect any known build configuration file
- **Expected behavior:** Verifier reports "No recognized build system detected" in `verifier.md`, skips build/test steps, and still performs acceptance criteria verification against `feature.md`
- **Must NOT:** Crash or halt the pipeline

### EC-3: Missing Agent for Per-Task Routing

- **Condition:** Task file specifies `agent: some-unknown-agent`
- **Expected behavior:** Orchestrator defaults to `implementer` and logs a warning in its output
- **Must NOT:** Fail the entire wave because one task has an unrecognized agent

### EC-4: Empty Wave After Sub-Wave Splitting

- **Condition:** Wave has 0 tasks (e.g., all tasks already completed in replan mode)
- **Expected behavior:** Orchestrator skips the empty wave and proceeds to the next step
- **Must NOT:** Hang or error on an empty wave

### EC-5: APPROVAL_MODE in Non-Interactive Environment

- **Condition:** `APPROVAL_MODE` is `true` but the environment doesn't support interactive pausing
- **Expected behavior:** Orchestrator should use whatever pause mechanism is available (e.g., print a message and wait for input). If no mechanism exists, log a warning and proceed autonomously
- **Must NOT:** Block indefinitely

### EC-6: Circular Dependencies in Tasks

- **Condition:** Task A depends on Task B, and Task B depends on Task A
- **Expected behavior:** Planner detects circular dependencies during planning and breaks the cycle (or reports ERROR if it cannot)
- **Must NOT:** Create an unresolvable execution plan

### EC-7: Security Issues Found During Implementation

- **Condition:** Implementer discovers a security vulnerability in existing code while working on their task
- **Expected behavior:** If the fix is within the task's scope (≤3 files, related to the task), fix it. If it exceeds scope, document it in the task file as a finding for the reviewer to flag
- **Must NOT:** Ignore security issues or expand scope uncontrollably

### EC-8: Verifier Finds Unit Test Failures

- **Condition:** Unit tests written by implementers fail during verification (should have passed during TDD)
- **Expected behavior:** Verifier flags this as a high-severity issue in `verifier.md`, noting that TDD should have caught it. The failure triggers the standard Verifier→Planner feedback loop
- **Must NOT:** Silently pass a build with failing tests

### EC-9: Decision Log Conflicts

- **Condition:** `decisions.md` contains a decision that conflicts with the current feature's design
- **Expected behavior:** Researcher notes the conflict in the analysis. Critical thinker flags it. The design should address whether to override the prior decision (with justification) or conform to it
- **Must NOT:** Silently ignore contradictory prior decisions

### EC-10: No Security Concerns Applicable

- **Condition:** A feature has no security implications (e.g., a purely cosmetic UI change)
- **Expected behavior:** Designer's security section states "No security considerations identified" with brief justification. Reviewer's security scan still runs (secrets/PII scan always applies) but OWASP deep review is skipped
- **Must NOT:** Omit the security section entirely or force unnecessary security analysis

---

## User Stories / Flows

### US-1: Standard Feature Development (Happy Path)

1. User provides a feature request
2. Orchestrator creates `initial-request.md` and dispatches 3 parallel researchers (max 4 cap not hit)
3. Researchers use hybrid retrieval (semantic_search → grep_search → read_file) and produce partial analyses with confidence metadata
4. Synthesis researcher merges partials into `analysis.md`, reads `decisions.md` if it exists
5. Spec agent creates `feature.md` with structured edge cases and testable acceptance criteria
6. Designer creates `design.md` with security considerations and failure/recovery sections
7. Critical thinker reviews design against structured risk categories, writes `design_critical_review.md`, returns `DONE:`
8. Planner detects initial mode, creates plan with task size limits enforced, adds pre-mortem analysis section
9. Orchestrator dispatches implementers in waves of ≤4. Each implementer follows TDD: write tests → fail → code → pass → refactor. Each runs `get_errors` after every edit
10. Verifier detects build system, runs full build + test suite, verifies acceptance criteria (integration focus)
11. Reviewer performs security-scoped review (secrets scan + OWASP if applicable), adds architectural decisions to `decisions.md`
12. Feature is complete

### US-2: Replan After Verification Failure

1. Verifier finds integration failures (unit tests pass, but cross-task interactions fail)
2. Orchestrator triggers Verifier→Planner feedback loop
3. Planner detects replan mode (verifier.md exists with failures)
4. Planner creates remediation tasks respecting task size limits
5. Implementers fix issues using TDD
6. Verifier re-runs (up to 3 total attempts)

### US-3: Feature with Documentation Tasks

1. Planner identifies that the feature needs API documentation
2. Planner creates a task with `agent: documentation-writer`
3. Orchestrator reads the `agent` field and dispatches to documentation-writer
4. Documentation-writer generates API docs without modifying source code

### US-4: Feature with Human Approval Gates

1. `APPROVAL_MODE` is set to `true` in the prompt
2. After research synthesis, orchestrator pauses for human review
3. Human approves → orchestrator proceeds to spec
4. After planning, orchestrator pauses again
5. Human approves → orchestrator proceeds to implementation

---

## Test Scenarios

### TS-1: File Existence Test

Verify all 11 output files exist in `NewAgentsAndPrompts/` and are non-empty.

### TS-2: Critical Thinker Bug Fixes

- Read `critical-thinker.agent.md` → search for `## Completion Contract` → must exist
- Read `critical-thinker.agent.md` → search for `design_critical_review.md` → must be specified as output

### TS-3: TDD Cluster Consistency

- Read `implementer.agent.md` → must contain TDD workflow steps (write test, confirm fail, write code, confirm pass)
- Read `implementer.agent.md` → must NOT contain "Do NOT build" or "Do NOT run tests"
- Read `implementer.agent.md` → must require `get_errors` after file edits
- Read `verifier.agent.md` → must reference "integration-level" verification
- Read `orchestrator.agent.md` → must state implementers do unit-level TDD
- Cross-check: no contradiction between the three files

### TS-4: Concurrency Cap Consistency

- Read `orchestrator.agent.md` → must specify "4" as concurrent limit
- Read `orchestrator.agent.md` → must describe sub-wave splitting for >4 tasks
- Read `feature-workflow.prompt.md` → must mention max 4 concurrent

### TS-5: Planner Completeness

- Read `planner.agent.md` → must contain "Pre-Mortem" or "pre-mortem" section
- Read `planner.agent.md` → must contain task size limits (3 files, 500 lines, 2 dependencies)
- Read `planner.agent.md` → must contain mode detection (initial, replan, extension)

### TS-6: Security Thread Coverage

- Read `implementer.agent.md` → must contain rules about secrets, API keys, PII
- Read `reviewer.agent.md` → must contain OWASP reference and secrets scanning
- Read `designer.agent.md` → must contain security considerations section

### TS-7: Technology Portability

- Read `verifier.agent.md` → must NOT contain "dotnet", "TourneyPal", ".sln" or any hardcoded project reference
- Read `verifier.agent.md` → must contain technology detection logic referencing multiple build systems

### TS-8: Anti-Drift Anchors

- For each of the 10 agent files: search for a final anchor block containing "Stay as" or "stay as" → must exist

### TS-9: Completion Contract Universality

- For each of the 10 agent files: search for `DONE:` and `ERROR:` in a completion contract section → must exist

### TS-10: Per-Task Agent Routing (Cluster B)

- Read `planner.agent.md` → must define `agent` field in task file format
- Read `orchestrator.agent.md` → must read `agent` field and dispatch accordingly
- Read `orchestrator.agent.md` → must default to `implementer` when `agent` field is absent

### TS-11: Approval Gates (Cluster C)

- Read `orchestrator.agent.md` → must contain conditional logic for `APPROVAL_MODE`
- Read `feature-workflow.prompt.md` → must define `{{APPROVAL_MODE}}`
- Verify default behavior is autonomous (no pause when APPROVAL_MODE is unset)

### TS-12: Documentation Writer Agent

- Read `documentation-writer.agent.md` → must define role, workflow, completion contract, anti-drift anchor
- Read `documentation-writer.agent.md` → must NOT modify source code
- Verify agent is NOT referenced in the core 8-step pipeline (only via per-task routing)

### TS-13: Hybrid Retrieval Strategy

- Read `researcher.agent.md` → must prescribe semantic_search, grep_search, read_file, file_search methodology

### TS-14: Cross-Agent Consistency

- No agent references a rule that contradicts another agent
- All agents using `DONE:`/`ERROR:` use the same format
- All reading guidance uses the same ~200 line recommendation consecutively

### TS-15: Error Handling Guidance

- Read at least 3 agent files → each must contain error handling categories (transient retry, persistent report, security flag)

---

## Dependencies & Risks

### Implementation Dependencies

#### Cluster A — TDD (MUST coordinate, 3 files)

- FR-1.1 (orchestrator global rule) + FR-7.1 (implementer TDD workflow) + FR-8.1 (verifier role redefinition)
- **All 3 must be implemented together.** If any one is omitted, the agents will contradict each other.

#### Cluster B — Per-Task Agent Routing (MUST coordinate, 2+ files)

- FR-1.4 (orchestrator routing) + FR-6.4 (planner agent field) + FR-10.3 (prompt rule)
- FR-11.1 (documentation-writer) depends on Cluster B being implemented

#### Cluster C — Approval Gates (MUST coordinate, 2 files)

- FR-1.6 (orchestrator pause logic) + FR-10.2 (prompt APPROVAL_MODE variable)

#### Sequential Dependencies

- FR-7.4 (implementer self-reflection) should follow FR-7.1 (TDD) so reflection can reference the TDD cycle
- FR-11.1 (documentation-writer) requires FR-1.4 + FR-6.4 (per-task routing)
- FR-6.2 (task size limits) ideally precedes FR-6.4 (smaller tasks benefit from routing)

#### Independent Changes (any order)

All other FRs are independently adoptable and have no ordering constraints.

### Risks

| Risk                                                                     | Likelihood | Impact | Mitigation                                                                                                          |
| ------------------------------------------------------------------------ | ---------- | ------ | ------------------------------------------------------------------------------------------------------------------- |
| TDD workflow slows implementer execution significantly                   | Medium     | Medium | TDD fallback for non-testable contexts (EC-1); implementer writes focused unit tests, not comprehensive test suites |
| Concurrency cap of 4 is too restrictive for large features               | Low        | Low    | Cap is a reasonable default; can be adjusted in future iterations                                                   |
| Task size limits (3 files, 500 lines) cause excessive task decomposition | Medium     | Low    | Limits are calibrated from Gem Team's production use; can be adjusted based on experience                           |
| Anti-drift anchors consume tokens without measurable benefit             | Low        | Low    | Anchors are 3-5 lines; minimal token cost                                                                           |
| `list_code_usages` tool not available in all environments                | Medium     | Low    | Fallback to `grep_search` specified in FR-7.3                                                                       |
| `APPROVAL_MODE` doesn't work in target prompt system                     | Low        | Medium | Feature is optional; autonomous mode is the default                                                                 |
| Technology detection in verifier misidentifies build system              | Low        | Medium | Priority-ordered detection list; fallback to error reporting                                                        |
| Shared operating rules (error handling) are too generic to be useful     | Low        | Low    | Rules are specific enough to guide behavior without being prescriptive                                              |

---

## What Was Evaluated and NOT Adopted

These items from the Gem Team were evaluated against the "make agents better" standard and rejected for concrete technical reasons:

| Feature                                         | Reason for Rejection                                                                                                                                                                                     |
| ----------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| YAML state file (`plan.yaml`)                   | Markdown artifacts are more readable, diffable, and consistent with Forge's artifact philosophy. No functional benefit justifies the format change.                                                      |
| Dynamic DAG dispatch                            | Forge's deterministic pipeline is genuinely easier to debug, predict, and reason about. Dynamic dispatch adds complexity without proportional benefit for the fixed-stage workflow.                      |
| JSON completion contracts                       | `DONE:`/`ERROR:` is simpler, human-readable, and consistent. JSON adds parsing complexity without functional benefit.                                                                                    |
| Three-state completion (`needs_revision`)       | Forge's existing retry loops (verifier→planner, reviewer→planner) already handle revision. The third state adds semantic value but the orchestrator's loop logic already covers the use case.            |
| MCP-based memory system                         | Requires infrastructure (MCP memory tools) not universally available. File-based `decisions.md` provides the core benefit without the dependency.                                                        |
| Chrome Tester agent                             | Requires Chrome DevTools MCP. Too specialized for the core agent package.                                                                                                                                |
| DevOps agent                                    | Infrastructure-specific (Docker, K8s, CI/CD). Adds complexity and mandatory approval gates that conflict with Forge's autonomous model.                                                                  |
| XML-like semantic tags (`<role>`, `<workflow>`) | Unvalidated hypothesis that XML tags improve LLM parsing. Markdown headings are more human-readable. The `<final_anchor>` anti-drift pattern is adopted as prose, not requiring the full XML tag system. |
| YAML research output format                     | Markdown is more readable and consistent with the artifact chain.                                                                                                                                        |
| `disable-model-invocation: true`                | Platform-specific frontmatter field. Forge achieves the same via explicit prose instructions.                                                                                                            |
| Shared "zero explanation" communication style   | May reduce tokens but also reduces artifact quality. Forge's rich markdown artifacts benefit from contextual explanation.                                                                                |
| Tool activation calls                           | Gem-specific environment pattern. Not applicable to Forge's runtime.                                                                                                                                     |
