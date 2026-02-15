# Impact Analysis — Agent Improvements

## Focus Area: impact

## Summary

All 9 Forge agent files and the prompt file require modifications; the biggest changes hit implementer (TDD enforcement), planner (pre-mortem + task size limits), verifier (role shift + technology-agnostic commands), and reviewer (security scanning); 3 agents (spec, designer, critical-thinker) need minor enhancements; orchestrator and prompt need concurrency cap + optional approval gates; one new optional agent (documentation-writer) is recommended.

---

## Per-Agent Impact Analysis

### 1. orchestrator.agent.md

#### Sections to Modify

| Section                     | Change                                                                                                                                                                                                         | Priority | Rationale                                                                   |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | --------------------------------------------------------------------------- |
| **Global Rules**            | Add concurrency cap: "Maximum 4 concurrent subagent invocations. If a wave contains more than 4 tasks, split into sub-waves of at most 4."                                                                     | P1       | Gem-orchestrator caps at 4. Unbounded parallelism risks context exhaustion. |
| **Global Rules**            | Update implementer rule from "Implementers never build or run tests" to "Implementers perform unit-level TDD (run tests for their task). All integration-level verification is the verifier's responsibility." | P0       | Required to align with TDD enforcement in implementer.                      |
| **Step 5.2**                | Add sub-wave splitting logic: "If the wave contains >4 tasks, partition into sub-waves of ≤4 tasks. Dispatch each sub-wave sequentially, waiting for completion before dispatching the next sub-wave."         | P1       | Operationalizes the concurrency cap.                                        |
| **Step 5.2**                | Add per-task agent routing: "For each task, read the `agent` field from the task file (default: `implementer`). Invoke the specified agent."                                                                   | P2       | Future-proofs for documentation-writer and other optional agents.           |
| **Documentation Structure** | Add `decisions.md` to artifact list as optional cross-feature decision log.                                                                                                                                    | P2       | Lightweight alternative to Gem's memory system.                             |

#### What NOT to Adopt from gem-orchestrator

- **Dynamic DAG dispatch** — Forge's fixed pipeline is a core strength; replacing it would fundamentally change the architecture and reduce predictability.
- **YAML state file (plan.yaml)** — Markdown artifacts are more human-readable, easier to diff/review, and consistent with Forge's artifact philosophy.
- **`disable-model-invocation: true`** — Forge achieves the same via explicit instructions ("Never modify code or docs directly").
- **Memory system** — Infrastructure-dependent (requires MCP memory tools); Forge's file-based artifacts serve the same purpose within a single feature run.
- **Mandatory human pauses** — Forge's autonomous operation is a deliberate value proposition. Make pauses optional (see prompt changes) but never mandatory.

#### Forge Strengths to Preserve

- Fixed 8-step pipeline with deterministic ordering
- Rich artifact trail (10+ documents per feature)
- Retry/error recovery model (max 3 loops)
- Strict subagent delegation (never writes code)

---

### 2. researcher.agent.md

#### Sections to Modify

| Section                            | Change                                                                                                                                                                                                                                                                        | Priority | Rationale                                                                                                                      |
| ---------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------ |
| **Focused Research Rules**         | Add new "Retrieval Strategy" subsection prescribing hybrid retrieval: (1) `semantic_search` for conceptual discovery, (2) `grep_search` for exact pattern matching, (3) merge + deduplicate, (4) `read_file` for detailed examination, (5) `file_search` to verify existence. | P1       | gem-researcher's hybrid retrieval produces more consistent, thorough research. Forge currently has no prescribed methodology.  |
| **Partial Analysis File Contents** | Add optional metadata fields: `confidence_level` (high/medium/low), `coverage_estimate` (qualitative), `gaps` (with impact assessment).                                                                                                                                       | P2       | gem-researcher tracks formal confidence metadata. Useful for downstream agents (planner, spec) to gauge research completeness. |
| **Focused Research Rules**         | Add context-efficient reading guidance: "Prefer targeted line-range reads; limit to ~200 lines per `read_file` call. Use `semantic_search` and `grep_search` for discovery before reading full files."                                                                        | P3       | Common Gem pattern across all agents; reduces context waste.                                                                   |

#### What NOT to Adopt from gem-researcher

- **YAML output format** — Markdown is more readable, consistent with Forge's entire artifact chain. The YAML schema is highly verbose and adds maintenance burden.
- **Memory integration** — Forge has no memory MCP; file artifacts suffice.
- **`tavily_search` restrictions** — Forge researcher should use whatever tools are available without arbitrary restrictions on external search.
- **Tool activation steps** — Not part of Forge's instruction pattern.

#### Forge Strengths to Preserve

- Three fixed focus areas (architecture, impact, dependencies)
- Separate synthesis mode as distinct invocation
- Markdown output format
- "No solutioning" rule
- Completion contract format (DONE:/ERROR:)

---

### 3. planner.agent.md

#### Sections to Add

| Section to Add                        | Content                                                                                                                                                                                                                                                                                                                                     | Priority | Rationale                                                                                                                                                         |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Pre-Mortem Analysis** (new section) | After creating task index and execution waves, planner MUST identify failure scenarios. For each high/medium priority task: ≥1 failure scenario with likelihood (low/medium/high), impact (low/medium/high/critical), and mitigation. Include `overall_risk_level` and key assumptions. Add corresponding format block to plan.md contents. | P0       | gem-planner does this; Forge has no structured risk assessment before execution. Critical-thinker reviews design but nobody reviews the plan for execution risks. |
| **Task Size Limits** (new section)    | Hard limits: max 3 files touched, max 2 dependencies, max 500 lines changed, max medium effort. If a task exceeds any limit, break into subtasks.                                                                                                                                                                                           | P1       | gem-planner enforces these. Without guardrails, Forge tasks can be too large, reducing parallelism and increasing blast radius.                                   |

#### Sections to Modify

| Section                    | Change                                                                                                                                                                                                                                                              | Priority | Rationale                                                                                                     |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | ------------------------------------------------------------------------------------------------------------- |
| **Task File Requirements** | Add optional `agent` field (default: `implementer`). Valid values: any agent defined in the workspace.                                                                                                                                                              | P2       | Enables per-task routing (e.g., documentation tasks to documentation-writer).                                 |
| **plan.md Contents**       | Add `Pre-Mortem Analysis` section specification (overall_risk_level, assumptions, task risk assessment table).                                                                                                                                                      | P0       | Documents the pre-mortem output format.                                                                       |
| **plan.md Contents**       | Add optional `Implementation Specification` section (code_structure, affected_areas, component_details, integration_points).                                                                                                                                        | P2       | gem-planner includes this; gives implementers clearer architectural guidance without reading design.md fully. |
| **Workflow**               | Add mode awareness at the top: detect if this is initial planning (no plan.md), replan (verifier.md with failures), or extension (existing plan with new objectives). Currently the planner handles replan via verifier report but doesn't explicitly detect modes. | P1       | gem-planner has explicit mode detection; makes replan and extension flows clearer.                            |
| **Rules for Dependencies** | Add YAGNI/KISS/DRY guidance: "Prefer simpler decompositions. Avoid unnecessary tasks. Each task should deliver tangible, testable value."                                                                                                                           | P2       | Cross-cutting principle from gem-planner.                                                                     |

#### What NOT to Adopt from gem-planner

- **YAML plan format** — Keep markdown for plan.md and individual task files. More readable and consistent with Forge.
- **Mandatory plan_review pause** — Forge is autonomous. Optional approval is handled at the prompt level.
- **Sequential IDs (task-001)** — Forge's numeric prefix (01-task-a) works fine.

#### Forge Strengths to Preserve

- Execution wave model (vs. pure DAG)
- Individual task files (vs. single plan.yaml)
- Markdown format throughout
- Dependency graph format in plan.md
- Completion contract (DONE: N tasks, M waves)

---

### 4. implementer.agent.md

**This agent requires the most significant changes.**

#### Sections to Replace/Modify

| Section                          | Change                                                                                                                                                                                                                                                                                                                              | Priority | Rationale                                                                                                                                |
| -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| **Workflow** (replace)           | Replace the current 5-step workflow with TDD workflow: (1) Read task spec, (2) Write failing tests FIRST, (3) Confirm tests FAIL, (4) Write MINIMAL code to pass tests, (5) Run `get_errors` after every file edit, (6) Confirm tests PASS, (7) Optional: refactor for clarity/DRY while keeping tests green, (8) Update task file. | P0       | gem-implementer's TDD cycle is its biggest advantage. Catches issues at implementation time instead of deferring everything to verifier. |
| **Rules** (modify significantly) | Add: "Run `get_errors` after EVERY file edit — do not batch error checking."                                                                                                                                                                                                                                                        | P0       | gem-implementer does this; prevents accumulation of compile/lint errors.                                                                 |
| **Rules** (add)                  | Add: "If tests fail after implementation, debug and fix before returning."                                                                                                                                                                                                                                                          | P0       | Part of TDD green phase.                                                                                                                 |
| **Rules** (add)                  | Add code quality principles: "Follow YAGNI/KISS/DRY. Write minimal, concise, modular code. Prefer functional programming patterns where appropriate."                                                                                                                                                                               | P1       | gem-implementer expertise section lists these explicitly.                                                                                |
| **Rules** (add)                  | Add security rules: "Never hardcode secrets, API keys, tokens, or passwords. Never expose PII in logs or error messages. Fix security vulnerabilities before returning."                                                                                                                                                            | P1       | gem-implementer has explicit security rules.                                                                                             |
| **Rules** (add)                  | Add: "Use `list_code_usages` before refactoring existing code to understand all call sites."                                                                                                                                                                                                                                        | P1       | gem-implementer requires this; prevents breaking unknown consumers.                                                                      |
| **Rules** (remove)               | Remove: "Do NOT build the project" and "Do NOT run tests". Replace with TDD instructions.                                                                                                                                                                                                                                           | P0       | Contradicts new TDD workflow.                                                                                                            |
| **Description** (frontmatter)    | Update from "Does not build or run tests" to "Implements one task using TDD with inline unit verification."                                                                                                                                                                                                                         | P0       | Reflects new role.                                                                                                                       |

#### Section to Add

| Section to Add                    | Content                                                                                                                                                                                                              | Priority | Rationale                                                      |
| --------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | -------------------------------------------------------------- |
| **Self-Reflection** (new section) | Before returning, verify: (1) Does output fully address the task? (2) Any obvious omissions/errors? (3) Would a senior engineer approve? Fix issues before returning. Triggered for medium/high priority tasks only. | P3       | gem-implementer's reflect step. Low priority but adds quality. |

#### What NOT to Adopt from gem-implementer

- **Reading plan.yaml** — Forge implementer MUST NOT read plan.md (strict isolation preserved).
- **JSON return format** — Keep DONE:/ERROR: completion contract.
- **Tool activation steps** — Not part of Forge's pattern.
- **Adherence to tech_stack from plan** — Forge implementer gets tech guidance from design.md instead.

#### Forge Strengths to Preserve

- Single task scope — one task only
- Strict input restrictions (no plan.md access)
- feature.md and design.md as inputs
- No unrelated changes, no inferred scope

---

### 5. verifier.agent.md

#### Sections to Modify

| Section                                         | Change                                                                                                                                                                                                                                                                                                                                       | Priority | Rationale                                                                                                 |
| ----------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | --------------------------------------------------------------------------------------------------------- |
| **Role** (modify)                               | Change from "sole agent responsible for building the project and running tests" to "integration-level verification agent. While implementers verify their individual tasks at the unit level via TDD, the verifier runs the full build, complete test suite, and cross-references all task acceptance criteria to catch integration issues." | P0       | Implementer now does unit-level TDD; verifier focuses on integration.                                     |
| **Workflow > Build** (modify)                   | Replace hardcoded `dotnet build TourneyPal.sln` with technology-agnostic instruction: "Identify the project's build system from the codebase (e.g., package.json scripts, Makefile, .sln, pom.xml, Cargo.toml, go.mod). Run the appropriate build command. Record all build errors and warnings with file paths and line numbers."           | P1       | Current verifier is hardcoded to a specific .NET project. Makes the agent reusable across any technology. |
| **Workflow > Test Execution** (modify)          | Replace hardcoded `dotnet test tests/TourneyPal.Tests` with: "Identify the project's test runner from the codebase. Run the full test suite. Record all test results: passing, failing, and skipped."                                                                                                                                        | P1       | Same hardcoding issue as build.                                                                           |
| **Workflow > Task-Level Verification** (modify) | Add: "Note: Implementers have already verified unit tests pass for their tasks. Focus on cross-task interactions, integration-level failures, and acceptance criteria that span multiple tasks."                                                                                                                                             | P0       | Clarifies the shifted verification boundary.                                                              |

#### What NOT to Adopt

- **Removing the verifier** — Gem has no dedicated verifier (inline per-task only). Forge's dedicated verifier is a clear architectural advantage for catching integration issues.
- **Per-task JSON handoff** — Keep comprehensive markdown report format.

#### Forge Strengths to Preserve

- Dedicated verifier agent (Forge's architectural advantage)
- Full build + full test suite execution
- Per-task verification against acceptance criteria
- Targeted re-verification support
- Comprehensive markdown report (verifier.md)
- Error mapping to specific task IDs for targeted re-planning

---

### 6. reviewer.agent.md

#### Sections to Add

| Section to Add                    | Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Priority | Rationale                                                                                                   |
| --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------- | ----------------------------------------------------------------------------------------------------------- |
| **Security Review** (new section) | Add after existing review workflow. For ALL reviews: scan for hardcoded secrets/API keys/tokens/passwords; check for PII in logs/errors/responses; verify input validation. For SECURITY-SENSITIVE changes (auth, data, payments, admin): OWASP Top 10 scan (injection, broken auth, sensitive data exposure, XXE, broken access control, misconfig, XSS, insecure deserialization, known vulnerabilities, insufficient logging); verify auth/authz on new endpoints; check for SQLi, XSS, CSRF. | P1       | gem-reviewer is security-first. Forge reviewer has zero explicit security focus. This is a significant gap. |
| **Review Depth** (new subsection) | Tiered depth: Full (high priority, security-sensitive, auth, PII, production) — complete security + quality scan; Standard (medium priority, feature additions) — secrets + basic OWASP + code quality; Lightweight (low priority, bug fixes, minor refactors) — secrets scan + naming/style only.                                                                                                                                                                                               | P2       | gem-reviewer tiers review depth. Prevents over-reviewing trivial changes and under-reviewing critical ones. |

#### Sections to Modify

| Section      | Change                                                                                                                                             | Priority | Rationale                                                                                                              |
| ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | ---------------------------------------------------------------------------------------------------------------------- |
| **Workflow** | Add explicit read-only enforcement: "The reviewer MUST NOT modify source code files. It produces review.md and updates .github/instructions only." | P1       | gem-reviewer is strictly read-only. Forge reviewer doesn't explicitly state this; adding clarity prevents scope creep. |
| **Workflow** | Add quality bar: "Apply the standard: Would a staff engineer approve this code?"                                                                   | P3       | gem-reviewer quality bar. Lightweight but sets clear expectation.                                                      |

#### What NOT to Adopt from gem-reviewer

- **Dropping code quality review** — gem-reviewer is security-only. Forge reviewer should KEEP its existing scope (maintainability, readability, naming, test quality, architecture) AND add security on top.
- **JSON handoff format** — Keep review.md markdown output.
- **Removing .github/instructions updates** — This is a Forge strength not present in Gem.

#### Forge Strengths to Preserve

- Full git diff code quality review (maintainability, readability, naming, test quality, architecture)
- .github/instructions updates based on codebase changes
- Blocking/non-blocking issue classification
- review.md with actionable items for orchestrator
- Completion contract (DONE:/ERROR:)

---

### 7. designer.agent.md

Gem Team has no dedicated designer agent. Changes are derived from cross-cutting Gem patterns.

#### Sections to Modify

| Section                | Change                                                                                                                                                                                                                | Priority | Rationale                                                                                                                          |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| **design.md Contents** | Add "Security Considerations" subsection: authentication/authorization patterns, data protection approach, threat model (if applicable), input validation strategy.                                                   | P2       | gem-implementer and gem-reviewer both emphasize security. If security is designed upfront, implementation and review are smoother. |
| **design.md Contents** | Add "Failure & Recovery" subsection: expected failure modes, retry/fallback strategies, graceful degradation approach.                                                                                                | P2       | Complements planner's pre-mortem by addressing failure at the design level.                                                        |
| **Workflow**           | Add brief self-verification step: "Before returning, verify the design addresses all functional and non-functional requirements from feature.md and that every acceptance criterion has a clear implementation path." | P3       | Inspired by gem-implementer's self-reflection pattern.                                                                             |

#### What NOT to Adopt

No specific gem-designer exists to draw from. Changes are enhancements from cross-cutting patterns.

#### Forge Strengths to Preserve

- All existing design.md structure and contents
- Linked inputs (analysis.md, feature.md)
- Tradeoffs and alternatives considered
- Testing strategy section
- Completion contract

---

### 8. spec.agent.md

Gem Team has no dedicated spec agent. Changes are minor enhancements from cross-cutting patterns.

#### Sections to Modify

| Section                              | Change                                                                                                                                                                                             | Priority | Rationale                                                                            |
| ------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | ------------------------------------------------------------------------------------ |
| **feature.md Contents > Edge Cases** | Add guidance for structured edge case documentation: each edge case should have clear input/condition, expected behavior, and severity if missed.                                                  | P3       | Improves testability of edge cases.                                                  |
| **Workflow**                         | Add brief verification step: "Before returning, verify all acceptance criteria are testable (have clear pass/fail definitions) and all functional requirements map to at least one test scenario." | P3       | Self-check to ensure spec quality. Inspired by gem-implementer's reflection pattern. |

#### Forge Strengths to Preserve

- Full feature.md structure (functional/non-functional requirements, constraints, acceptance criteria, edge cases, user stories, test scenarios)
- Grounding in initial-request.md
- "Perform minimal additional research" instruction
- Completion contract

---

### 9. critical-thinker.agent.md

Gem Team has no dedicated critical-thinker agent. This is Forge's most distinctive agent.

#### Sections to Add

| Section to Add                         | Content                                                                                                                                                                                                                           | Priority | Rationale                                                                                                                    |
| -------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | ---------------------------------------------------------------------------------------------------------------------------- |
| **Output Specification** (new section) | Add explicit output: "Produce `docs/feature/<feature-slug>/design_critical_review.md` containing: identified risks, challenged assumptions, alternative approaches considered, questions raised, and recommended design changes." | P1       | Orchestrator references this file but the critical-thinker agent doesn't specify it. Creates ambiguity.                      |
| **Completion Contract** (new section)  | Add: "Return exactly one line: DONE: <summary of findings> or ERROR: <reason>."                                                                                                                                                   | P1       | ALL other Forge agents have completion contracts. The critical-thinker is the only one missing this, creating inconsistency. |

#### Sections to Modify

| Section          | Change                                                                                                                                                                                | Priority | Rationale                                                                                                                                                                            |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Instructions** | Add structured risk categories to probe: security implications, scalability concerns, maintainability risks, backwards compatibility, edge cases not covered, performance under load. | P2       | Currently the instructions are generic ("challenge assumptions"). Adding specific risk categories ensures consistent coverage. Complements planner's pre-mortem at the design level. |
| **Instructions** | Add: "Your review should ground risks in specific technical details from design.md rather than abstract concerns."                                                                    | P2       | Prevents vague, unhelpful criticism.                                                                                                                                                 |

#### What NOT to Adopt

No Gem equivalent exists. The critical-thinker is unique to Forge and should be enhanced rather than replaced.

#### Forge Strengths to Preserve

- Devil's advocate approach
- Question-driven methodology (ask "Why?")
- "Do not suggest solutions" rule (forces deep thinking)
- One question at a time discipline
- Strong opinions, loosely held

---

### 10. feature-workflow.prompt.md

#### Sections to Add

| Section to Add                            | Content                                                                                                                                                                                                                                                               | Priority | Rationale                                                                                                                              |
| ----------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| **Optional Approval Gates** (new section) | Add: "If `{{APPROVAL_MODE}}` is `true`: pause after research synthesis (step 1.2) and present analysis.md summary; pause after planning (step 4) and present plan.md for approval. Continue only after user confirms. If not set or `false`: run fully autonomously." | P2       | Gem-orchestrator has 3 mandatory pauses. Forge should offer this as opt-in for high-stakes projects without breaking default autonomy. |

#### Sections to Modify

| Section   | Change                                                                                                                    | Priority | Rationale                                                          |
| --------- | ------------------------------------------------------------------------------------------------------------------------- | -------- | ------------------------------------------------------------------ |
| **Rules** | Add: "Limit to 4 concurrent subagent invocations. Split waves into sub-waves as needed."                                  | P1       | Reinforces the orchestrator's concurrency cap at the prompt level. |
| **Rules** | Add: "If an implementer task file specifies an `agent` field, dispatch to that agent instead of the default implementer." | P2       | Supports per-task agent routing.                                   |

#### Forge Strengths to Preserve

- `{{USER_FEATURE}}` variable
- Autonomous by default
- All existing rules (display step, display subagent, never implement code, retry failed steps, dispatch in parallel)

---

### 11. New Agents Assessment

#### Documentation Writer — RECOMMENDED as Optional (P2)

**Rationale:** gem-documentation-writer fills a real gap. Forge produces workflow artifacts (analysis, spec, design, plan, review) but no end-user or API documentation. For projects with documentation requirements, a documentation-writer agent would:

- Generate API specs (OpenAPI/Swagger)
- Produce architectural diagrams (Mermaid)
- Maintain code-documentation parity
- Be routed via the planner's `agent` field on documentation tasks

**Impact on existing files:**

- `planner.agent.md`: Already gets `agent` field support (see above)
- `orchestrator.agent.md`: Already gets per-task agent routing (see above)
- New file: `documentation-writer.agent.md` in NewAgentsAndPrompts/

**NOT in core pipeline** — invoked only when planner assigns documentation tasks.

#### Chrome Tester — NOT RECOMMENDED for Core (P3)

**Rationale:** gem-chrome-tester requires Chrome DevTools MCP tools that may not be available in all environments. Too specialized for the core pipeline.

**If added:** Should be entirely optional, invoked via `agent` field on browser-testing tasks. Would require Chrome MCP as a prerequisite.

#### DevOps — NOT RECOMMENDED (Skip)

**Rationale:** Too infrastructure-specific (Docker, K8s, CI/CD). Most projects using Forge don't need deployment automation from the agent framework. The approval gates (production deployment approval) add complexity that conflicts with Forge's autonomous operation model.

---

## Cross-Cutting Improvements (Apply to ALL Agents)

### 1. Completion Contract Consistency (P1)

**Gap:** `critical-thinker.agent.md` is the ONLY agent without a DONE:/ERROR: completion contract. Add one.

**Impact:** 1 file changed.

### 2. Context-Efficient File Reading (P3)

**Pattern from Gem:** "Prefer semantic search, file outlines, and targeted line-range reads; limit to ~200 lines per read."

**Applies to:** researcher, planner, critical-thinker, verifier, reviewer — any agent that reads the codebase.

**Impact:** Add guidance to 5 files' rules/instructions.

### 3. Security Awareness (P1)

**Pattern from Gem:** Security is threaded through implementer, reviewer, and planner — not just the reviewer.

**Already covered by:**

- Implementer: security rules (no secrets/PII, fix vulnerabilities)
- Reviewer: security review section (OWASP, secrets scanning)
- Designer: security considerations in design.md

### 4. Self-Reflection Pattern (P3)

**Pattern from Gem:** Brief self-check before returning for medium/high priority work.

**Already covered by:** Implementer self-reflection section. Could optionally extend to reviewer, verifier.

---

## Gem Features That Should NOT Be Adopted

| Feature                              | Why Not                                                                                                                                                                                                                       |
| ------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **YAML state file (plan.yaml)**      | Markdown artifacts are core to Forge's identity — more readable, diffable, and consistent. YAML adds schema complexity without clear benefit for a fixed pipeline.                                                            |
| **Dynamic DAG dispatch**             | Forge's fixed pipeline (setup→research→spec→design→review→plan→implement→verify→review) is deterministic and predictable. Switching to dynamic dispatch would be a fundamental architecture change with marginal benefit.     |
| **Mandatory human pauses**           | Forge's autonomous operation is a deliberate design choice. Optional gates (via APPROVAL_MODE) are sufficient.                                                                                                                |
| **Memory system**                    | Requires MCP memory tools not available in all environments. File artifacts serve the same purpose within a feature run. A lightweight decision log (decisions.md) provides cross-feature persistence without infrastructure. |
| **`disable-model-invocation: true`** | Forge achieves the same outcome with explicit instructions. This is a Gem-specific mechanism.                                                                                                                                 |
| **Tool activation calls**            | Gem agents call `activate_web_interaction`, `activate_vs_code_interaction`, etc. This is an environment-specific pattern not applicable to Forge.                                                                             |
| **JSON handoff format**              | Forge's DONE:/ERROR: one-line contracts are simpler, more readable, and consistent across all agents.                                                                                                                         |
| **Chrome Tester as core agent**      | Requires Chrome DevTools MCP. Not universally available. Optional at best.                                                                                                                                                    |
| **DevOps agent**                     | Too infrastructure-specific. Adds complexity (approval gates, Docker, K8s) that conflicts with Forge's streamlined pipeline.                                                                                                  |

---

## Forge Strengths That MUST Be Preserved

1. **Fixed deterministic pipeline** — predictable 8-step workflow that's easy to reason about and debug
2. **Dedicated spec stage** — formal requirements document before design
3. **Dedicated design stage** — deep technical design with tradeoffs
4. **Critical-thinker gate** — adversarial design review before planning (unique to Forge)
5. **Dedicated verifier agent** — single authoritative integration verification point
6. **Synthesis step** — dedicated research merging prevents duplication
7. **Strict agent isolation** — clear I/O boundaries per agent
8. **Full autonomy by default** — runs end-to-end unattended
9. **Rich artifact trail** — 10+ human-readable markdown documents per feature
10. **DONE:/ERROR: completion contracts** — consistent, simple agent communication
11. **Markdown throughout** — all artifacts are markdown, not YAML/JSON

---

## File References

| File                                         | Changes Needed                                                                              | Priority |
| -------------------------------------------- | ------------------------------------------------------------------------------------------- | -------- |
| `.github/agents/orchestrator.agent.md`       | Concurrency cap, implementer role update, per-task routing, decisions.md                    | P0-P2    |
| `.github/agents/researcher.agent.md`         | Hybrid retrieval strategy, confidence metadata, efficient reading                           | P1-P3    |
| `.github/agents/planner.agent.md`            | Pre-mortem analysis, task size limits, agent field, mode detection, implementation spec     | P0-P2    |
| `.github/agents/implementer.agent.md`        | TDD workflow, get_errors, YAGNI/KISS/DRY, security rules, list_code_usages, self-reflection | P0-P3    |
| `.github/agents/verifier.agent.md`           | Role shift to integration-level, remove hardcoded dotnet commands, technology-agnostic      | P0-P1    |
| `.github/agents/reviewer.agent.md`           | Security review section, tiered depth, read-only enforcement, quality bar                   | P1-P3    |
| `.github/agents/designer.agent.md`           | Security considerations, failure/recovery section, self-verification                        | P2-P3    |
| `.github/agents/spec.agent.md`               | Edge case structure, self-verification                                                      | P3       |
| `.github/agents/critical-thinker.agent.md`   | Output specification, completion contract, risk categories, grounded feedback               | P1-P2    |
| `.github/prompts/feature-workflow.prompt.md` | Optional approval gates, concurrency cap, per-task routing                                  | P1-P2    |
| (NEW) `documentation-writer.agent.md`        | Full new agent definition — API docs, diagrams, parity enforcement                          | P2       |

---

## Assumptions & Limitations

1. **TDD feasibility assumed** — TDD enforcement assumes all projects have a testable structure and that test runners are available in the agent's execution environment. For projects without test infrastructure, the implementer would need a fallback mode.
2. **Concurrency cap value (4) is a reasonable default** — The exact number may need tuning per environment. Gem uses 4; this is reasonable for most setups.
3. **Technology-agnostic verifier is feasible** — Replacing hardcoded dotnet commands assumes the verifier can detect build/test tooling from the codebase. This should work for standard projects but may need guidance for exotic setups.
4. **Documentation-writer as optional** — Assumes most Forge users don't need automated documentation generation as part of the core pipeline. If this is wrong, it should be integrated as a post-review step.
5. **No access to Gem's memory MCP tools** — Forge environments may not have `memory create/update/search` tools. The decision log convention (decisions.md) is a file-based alternative.
6. **Existing comparison docs were largely accurate** — Independent analysis confirmed the 10 improvements identified in `optimization-from-gem-team.md`. However, several gaps were found (see Open Questions).

## Open Questions

1. **TDD fallback for non-testable projects** — What should the implementer do if the project has no test framework or the task doesn't warrant tests? Should there be a `skip_tdd` flag in task files?
2. **Verifier technology detection** — Should the verifier use a heuristic (scan for package.json, .sln, Makefile) or should the planner/designer specify build/test commands in plan.md or design.md?
3. **Critical-thinker output format** — The orchestrator references `design_critical_review.md` but the critical-thinker agent never specifies this output. Is the file name correct? Should the format be structured (sections) or free-form?
4. **Decision log maintenance** — If decisions.md is adopted, who writes to it (reviewer?) and who reads it (researcher?). What's the lifecycle across multiple features?
5. **APPROVAL_MODE scope** — Should approval gates also pause after verification (before review), or only after research and planning?
6. **Implementation specification in plan.md** — The gem-planner includes code_structure, affected_areas, component_details. Does this overlap too much with Forge's design.md? Should it be a simplified reference to design.md instead?
7. **Documentation-writer trigger** — If adopted, should the documentation-writer run after implementation (post-verify) or should it be a taskable agent during implementation waves?
8. **Missing from existing analysis: Gem's `list_code_usages` pattern** — The optimization doc doesn't mention this gem-implementer practice. It should be evaluated: how widely available is `list_code_usages` as a tool, and should it be a MUST or SHOULD?
9. **Missing from existing analysis: Gem's communication style** — All Gem agents have a strict communication rule: "Output ONLY the requested deliverable. Zero explanation, zero preamble, zero commentary." Should Forge adopt this to reduce token waste?
