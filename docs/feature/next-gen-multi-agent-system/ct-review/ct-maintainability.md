# Critical Review: Maintainability

**Feature:** Next-Gen Multi-Agent System  
**Design:** Synthesized A+D+B architecture — 9 agents, typed YAML, zero-merge memory, SQL/YAML verification  
**Reviewer:** ct-maintainability  
**Date:** 2026-02-26

---

## Findings

### [Severity: High] Orchestrator Accumulates God-Agent Responsibilities

- **What:** The orchestrator is responsible for at least 9 distinct concerns: pipeline state management (pipeline manifest), schema validation of ALL agent outputs, evidence gate evaluation (COUNT checks across SQL or YAML tracks), dispatch routing (decision table with 15+ input/condition/action rows), risk assessment (determining reviewer count from risk summaries), approval gate management (interactive vs. autonomous, structured prompts), error categorization and retry orchestration, telemetry accumulation, and SQL availability detection. The existing Forge orchestrator is already 544 lines (per A-7 in feature.md) with fewer responsibilities — it doesn't do schema validation or evidence gating. Each new responsibility adds conditional branches the orchestrator prompt must handle correctly. NFR-6 states "changing one agent's behavior MUST NOT require changing other agents" — but modifying any agent's output schema requires updating the orchestrator's validation logic, routing table, and potentially evidence gates.
- **Where:** design.md §Decision 2 (Orchestrator agent detail), §Decision 3 (Orchestrator Decision Table at end of file), §Decision 4 (Schema Validation Protocol), §Decision 6 (Evidence Gating Mechanism)
- **Likelihood:** High — this is structural; every integration point flows through the orchestrator
- **Impact:** High — orchestrator bugs or context drift affect every pipeline step; debugging requires understanding all 9 concerns simultaneously
- **Assumption at risk:** That a single LLM agent definition can reliably execute 9 distinct responsibilities without context drift, and that prompt-level instructions are sufficient to enforce all validation/gating/routing logic consistently

### [Severity: High] 14 Schema Definitions in One File Without Evolution Strategy

- **What:** The design specifies 14 typed YAML schemas in a single `schemas.md` reference document. All 9 agents and the orchestrator reference this file. The only versioning mechanism is `schema_version: "1.0"` in agent output headers. There is no defined: (a) schema evolution process, (b) backward compatibility policy, (c) explicit dependency graph showing which agents produce/consume which schemas, (d) schema owner or change authority, (e) migration strategy when schemas change. schemas.md becomes the highest-coupling artifact in the entire system — a single change to one schema requires verifying all consumers still comply. For a "lean" system optimized to minimize coupling, this is a single point of failure for maintainability. The design also does not specify how the 14 schemas will be organized within the file (alphabetical? by pipeline step? by producer?), which affects discoverability for developers modifying them.
- **Where:** design.md §Schemas Reference, §Decision 4 (Communication & State), §Decision 10 (Implementation Checklist item #1 — schemas.md is P0)
- **Likelihood:** High — schemas will need to evolve as the system is used and edge cases appear
- **Impact:** High — schema changes without a defined process will cause cascading agent failures or silent data loss
- **Assumption at risk:** That 14 schemas can be defined correctly upfront as "1.0" and remain stable through implementation, testing, and real-world use without a formal evolution mechanism

### [Severity: High] Prompt-Level Schema Validation Undermines the Typed Communication Foundation

- **What:** The entire architecture is predicated on typed YAML providing reliable inter-agent communication (article principle #1). However, design.md §Decision 4 explicitly acknowledges: "Validation is prompt-level (the orchestrator is instructed to check required fields), not runtime-enforced. This is a pragmatic concession to the VS Code agent framework's capabilities." This means: (a) schema validation depends on the LLM orchestrator following instructions correctly every time, (b) there is no runtime guarantee that malformed YAML is caught, (c) the orchestrator could silently pass invalid data downstream, (d) COUNT-based evidence gating — the core mechanism preventing hallucinated verification — is itself dependent on the LLM accurately counting records. The design replaces Forge's prose-based routing with typed-schema-based routing, but the enforcement layer is the same: LLM prompt compliance. The typed schemas add structural value for agent output generation but the validation chain is only as strong as its weakest link, which remains an LLM interpreting instructions.
- **Where:** design.md §Decision 4 (Schema Validation Protocol, paragraph after step 6), §Decision 6 (Evidence Gating Mechanism — all gates are orchestrator-evaluated), Threat Model (first row: "LLM hallucinating verification results")
- **Likelihood:** Medium — LLMs are generally good at following structured validation instructions, but reliability degrades with context length and complexity
- **Impact:** High — a single schema validation miss could route invalid data through the pipeline undetected; evidence gate miscounts could approve unverified code
- **Assumption at risk:** That prompt-level validation by an LLM is sufficiently reliable to serve as the enforcement layer for a system whose core design principle is "typed schemas make inter-agent communication reliable"

### [Severity: Medium] Unified Verifier is Four Agents Behind a Single Interface

- **What:** The Verifier replaces V-Build, V-Tests, V-Tasks, and V-Feature. Internally it must: (a) execute a 3-tier verification cascade with multiple checks per tier, (b) manage an evidence ledger on either SQL or YAML track (code-switching based on a flag), (c) capture and compare baseline-vs-after states for regression detection, (d) verify acceptance criteria against spec-output.yaml, (e) dynamically discover available tooling for Tier 2 ("discover dynamically"), (f) handle tier3-infeasible gracefully. The design rates this 🟡 and says "bounded scope (tool execution, not reasoning)." But regression detection (comparing baseline to after, identifying which checks regressed) IS reasoning. Acceptance criteria verification against spec IS reasoning. Dynamic tool discovery IS unbounded. The Verifier is essentially an internal state machine (which tier? which track? did build fail so skip later tiers? how many signals collected so far?) living inside a single prompt. This is the definition of a mega-agent that the Direction A evaluation warned about.
- **Where:** design.md §Decision 6 (Verification Architecture), §Decision 2 (Verifier agent detail — tools, inputs, outputs, cascade), feature.md §Direction A risk: "unified verifier is complex"
- **Likelihood:** Medium — the Verifier's complexity depends on feature size and tooling availability; simple features may be fine
- **Impact:** High — Verifier failures or missed checks undermine the entire evidence-gating system; debugging a failed verification cascade requires tracing through 3 tiers × 2 tracks × baseline/after phases
- **Assumption at risk:** That consolidating 4 specialized agents into 1 with "bounded scope" actually reduces complexity rather than hiding it inside a single prompt

### [Severity: Medium] Adversarial Reviewer Parameterization Creates Hidden Mega-Prompt

- **What:** The Adversarial Reviewer replaces 7 Forge agents (4 CT + 3 R-agents) via parameterization: `review_scope: design | code`, `model: 3 options`, `risk_level: 3 options`. This creates 2 × 3 × 3 = 18 potential configurations. The fundamental issue: design review and code review are fundamentally different tasks. Design review examines architecture decisions in `design-output.yaml` + `design.md`. Code review examines `git diff --staged` + implementation reports + verification evidence. These require different evaluation criteria, different expertise, different tool usage (code review uses `run_in_terminal` for git diff; design review doesn't need terminal access). A single prompt must be general enough to handle both modes while being specific enough to catch the nuanced issues that specialized agents (ct-security, ct-maintainability, r-quality) were designed to find. The design says "simpler than Forge's 4-agent CT cluster" — but simplicity in agent count is not the same as simplicity in prompt content.
- **Where:** design.md §Decision 7 (Adversarial Review Design — Parameterization section), §Decision 2 (Adversarial Reviewer agent detail — note the different tool sets and inputs for design vs. code mode)
- **Likelihood:** Medium — the prompt will work for obvious issues but may miss subtle category-specific concerns (e.g., the nuanced maintainability analysis that a dedicated ct-maintainability agent performs)
- **Impact:** Medium — reduced review quality for specific concern categories; security issues in particular may suffer from diluted attention in a generalized prompt
- **Assumption at risk:** That a single parameterized prompt can match the review quality of 7 specialized prompts across all concern categories

### [Severity: Medium] Pipeline Manifest Becomes the New Growing Shared State

- **What:** The design eliminates memory.md and its merge overhead but replaces it with pipeline-manifest.yaml, which tracks: feature metadata, current step, all step statuses, ALL dispatch records (agent, instance, model, status, output_path, retry_count) for every dispatch, evidence gate statuses, approval logs, and error logs. For a typical 6-task feature with 2 verification iterations: ~22 dispatches × dispatch records + 6 evidence gates + 2 approval logs + potential error entries = easily 100+ YAML lines. For a complex feature (10 tasks, 3 iterations, 3 reviewers × 2 phases): the manifest could reach 300+ lines. Every agent reads this manifest for orientation (per the memory-first reading pattern: "Agent reads pipeline manifest for pipeline state, completed steps, output paths"). The zero-merge model trades write complexity (merge operations) for read complexity (every agent parsing an ever-growing manifest). This is the same shared-state scaling problem in a different format.
- **Where:** design.md §Pipeline Manifest Schema (final section), §Decision 5 (Memory-First Reading Pattern — Preserved), §Decision 3 (Dispatch Count Analysis — 18-22 dispatches)
- **Likelihood:** Medium — manifest growth is proportional to feature complexity; simple features stay small
- **Impact:** Medium — excessive manifest size means agents waste context window reading dispatch records from previous steps they don't care about; later agents (Knowledge Agent) receive the largest manifest
- **Assumption at risk:** That the pipeline manifest will remain small enough for efficient agent orientation, even for complex features with multiple iterations and retries

### [Severity: Medium] Dual-Track SQL/YAML Verification Doubles the Maintainable Surface

- **What:** Every verification operation must be implemented in both SQL and YAML. Evidence gates have both SQL queries AND YAML equivalents (design.md §Decision 6 Evidence Gating table has both columns). The Verifier must code-switch based on `verification_backend: sql | yaml`. The orchestrator must evaluate gates differently per track. When schemas evolve, both tracks must change in sync. No mechanism ensures parity between tracks. SQL has transactional atomicity (INSERT succeeds or fails atomically); YAML append-only doesn't (partial writes are possible). SQL supports `COUNT(*)` queries natively; YAML requires parsing and counting records. Edge cases may manifest differently: SQL constraint violations vs. YAML format errors. The design acknowledges this adds "implementation complexity" but positions it as "necessary for portability." The portability benefit is real, but the maintainability cost of keeping two tracks synchronized is not adequately addressed.
- **Where:** design.md §Decision 6 (SQL Schema, YAML Schema, Evidence Gating Mechanism table), §Failure & Recovery (Graceful Degradation — "SQL → YAML fallback"), feature.md EC-4 (SQLite unavailable)
- **Likelihood:** Medium — the YAML fallback will get less testing and attention than the SQL primary path
- **Impact:** Medium — bugs in one track may go unnoticed if testing primarily uses the other track; parity violations could produce different verification outcomes depending on environment
- **Assumption at risk:** That the two verification tracks can be kept in reliable sync without a parity-testing mechanism or shared abstraction layer

### [Severity: Medium] Agent Independence Is Limited by Orchestrator Coupling

- **What:** NFR-6 requires agents be "independently updateable" — changing one agent's behavior MUST NOT require changing others if output schemas remain compatible. However, the orchestrator contains hard-coded knowledge of: (a) which agents exist and their names, (b) what completion states each can return (orchestrator never returns NEEDS_REVISION), (c) routing table mapping specific agent outputs to specific next agents, (d) evidence gate thresholds per task type, (e) reviewer count logic tied to risk levels, (f) retry budgets per agent type. Adding, removing, or restructuring any agent requires updating the orchestrator's decision table (15+ rows), potentially the pipeline manifest schema, and schemas.md. The "independence" is only within each agent's prompt body — the integration surface is concentrated in the orchestrator. This is a known cost of centralized orchestration, but the design doesn't acknowledge the maintenance burden of the orchestrator-as-integration-hub.
- **Where:** design.md §Decision 2 (all 9 agent details with tools/inputs/outputs), Orchestrator Decision Table (15+ rows of routing logic), Pipeline Manifest Schema (hard-coded step structure)
- **Likelihood:** High — agents will need to change as the system is used and refined
- **Impact:** Medium — every agent change requires orchestrator review; the orchestrator becomes a bottleneck for changes, not just for execution
- **Assumption at risk:** That typed schemas at agent boundaries provide meaningful independence when the orchestrator hard-codes the routing table, evidence gates, and retry logic for each specific agent

### [Severity: Low] Human-Readable Companion Files Create Silent Drift Risk

- **What:** Agents produce both typed YAML (primary, used for routing) and Markdown companions (for humans). Nothing prevents agents from writing companions that are inconsistent with the YAML. Humans reading design.md may get different information than the orchestrator routing on design-output.yaml. No reconciliation mechanism exists. No validation checks companion-to-YAML consistency. Over time, as agents are revised and re-run, companions may accumulate stale information while YAML is regenerated correctly.
- **Where:** design.md §Decision 4 (Human-Readable Companions section)
- **Likelihood:** Medium — LLMs generating two outputs from the same data may diverge on nuance
- **Impact:** Low — routing decisions are unaffected; only human understanding is at risk
- **Assumption at risk:** That LLM-generated YAML and Markdown from "the same data" will always be substantively consistent

### [Severity: Low] No Documented Agent Addition/Removal Playbook

- **What:** The design specifies 9 agents and a clear agent definition template (Decision 10). But there is no documented process for: how to add a 10th agent (which files to update: orchestrator routing table, schemas.md definition, pipeline manifest schema, dispatch-patterns.md, testing strategy), how to remove an agent, how to split an agent that has grown too complex (likely the Verifier), or what testing is required for integration verification of a new agent. The design's claim of extensibility ("clear contracts enable agent-level updates" in Direction D assessment) is aspirational without a concrete playbook.
- **Where:** design.md §Decision 10 (Agent Definition Structure — shows template but not integration checklist), §Decision 2 (no extensibility process)
- **Likelihood:** Medium — 9 agents is an initial design; evolution is likely
- **Impact:** Low — experienced developers can infer the process from the design, but new team members will waste time discovering integration points
- **Assumption at risk:** That the agent architecture is only ever maintained by developers who were present during initial design

---

## Cross-Cutting Observations

- **Scalability concern (ct-scalability scope):** For complex features (10+ tasks, 50+ files per NFR-3), the pipeline manifest, verification ledger, and number of dispatch records could grow substantially. The design's concurrency cap (4) and wave partitioning handle execution scaling, but the state artifact size scaling is unaddressed.

- **Security concern (ct-security scope):** Prompt-level schema validation means a security blocker in an adversarial review verdict could theoretically be missed if the orchestrator LLM fails to parse the verdict correctly. The "security Blocker from ANY model → pipeline ERROR" policy (Decision 7) is only as reliable as the orchestrator's ability to read and evaluate the typed verdict schema.

- **Strategy concern (ct-strategy scope):** The 1588-line design document itself is a maintainability risk. Implementers must absorb a massive specification to work on any single agent. The design's Implementation Checklist (14 files, P0/P1/P2 priority) helps, but there's no guidance on which design sections are relevant to which implementer tasks. The design could benefit from a "reading guide" mapping implementation tasks to design sections.

- **Testing concern (ct-strategy scope):** The testing strategy section maps acceptance criteria to test types but doesn't address the fundamental question: who executes these tests? The tests are described as verification steps for the pipeline, but the pipeline is a set of prompt files — there's no test runner. Integration and E2E tests require simulating agent outputs, which means building test harnesses outside the agent framework.

---

## Requirement Coverage

| Requirement                               | Coverage Status              | Notes                                                                                                   |
| ----------------------------------------- | ---------------------------- | ------------------------------------------------------------------------------------------------------- |
| CR-1: Deterministic Pipeline Execution    | Adequate                     | Pipeline structure is deterministic; decision table is well-defined                                     |
| CR-2: Three-State Completion Contracts    | Adequate                     | All 9 agents have contracts; schema defined                                                             |
| CR-3: Parallel Subagent Dispatch          | Adequate                     | Pattern A, concurrency cap of 4, sub-wave partitioning                                                  |
| CR-4: Adversarial Multi-Model Review      | Adequate but complexity risk | Parameterized single agent creates hidden complexity (Finding 5)                                        |
| CR-5: Evidence Gating                     | At Risk                      | COUNT-based gates depend on prompt-level validation (Finding 3)                                         |
| CR-6: Risk Classification                 | Adequate                     | Per-file classification, escalation rules clear                                                         |
| CR-7: Justification Scoring               | Adequate                     | Decision-record schema with confidence levels defined                                                   |
| CR-8: Structured Approval Mode            | Adequate                     | Two gates, structured prompts, autonomous/interactive modes                                             |
| CR-9: No Pending Step                     | Adequate                     | All steps execute or skip deterministically                                                             |
| CR-10: Output Directory Structure         | Adequate                     | NewAgents/.github/ layout well-defined                                                                  |
| CR-11: Self-Verification                  | Adequate                     | Template includes self-verification section                                                             |
| CR-12: Anti-Drift Anchors                 | Adequate                     | Template includes anchor; all agent definitions will have it                                            |
| CR-13: Unified Severity                   | Adequate                     | Blocker/Critical/Major/Minor; severity-taxonomy.md reference                                            |
| CR-14: Pushback System                    | Adequate                     | Spec agent includes pushback evaluation                                                                 |
| CR-15: Bounded Retry Budget               | Adequate                     | All loops bounded; worst-case analysis provided                                                         |
| NFR-5: Context Efficiency                 | At Risk                      | Zero-merge trades write savings for read overhead; manifest grows (Finding 6)                           |
| NFR-6: Maintainability                    | At Risk                      | Agent independence limited by orchestrator coupling (Finding 8); schema evolution undefined (Finding 2) |
| FR-3.4: Orchestrator reads state directly | Adequate                     | Pipeline manifest is directly readable                                                                  |
| FR-6.3: No memory merge subagent          | Adequate                     | Zero-merge model eliminates merge dispatches                                                            |
