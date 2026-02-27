# Critical Review: Strategy

## Findings

### [Severity: Critical] Pipeline Manifest Write Contradiction

- **What:** The design describes the pipeline manifest as "orchestrator direct write" (Decision 2, Orchestrator outputs: "Pipeline manifest (YAML, orchestrator direct write)"), but the orchestrator's tool restrictions explicitly prohibit `create_file` and `replace_string_in_file`. The orchestrator MUST NOT modify files — all writes are delegated to subagents via `runSubagent`. This means the core state management artifact (the pipeline manifest) either: (a) requires the orchestrator to violate its own tool restrictions, or (b) requires a subagent dispatch for every manifest update (~8+ per pipeline run), which significantly inflates the dispatch count.
- **Where:** [design.md](../design.md) §Decision 2 (Orchestrator outputs) vs §Decision 2 (Orchestrator tool restrictions); also affects §Decision 5 (Memory System Redesign) and §Decision 3 (Dispatch Count Analysis)
- **Likelihood:** High — this is a structural contradiction, not an edge case
- **Impact:** High — if resolved via subagent dispatches for manifest updates, the "18-22 dispatches" claim inflates to ~26-30, significantly narrowing the gap with Forge's ~32+. If resolved by granting the orchestrator write tools, this breaks the read-only-coordinator pattern that isolates the orchestrator from accidental file corruption.
- **Assumption at risk:** That the orchestrator can maintain the pipeline manifest without file-write tools. The existing Forge orchestrator has the identical restriction ("MUST NOT use create_file, replace_string_in_file") and delegates ALL writes via `runSubagent`, including memory.md creation/updates.

---

### [Severity: Critical] YAML Schema Validation Is Prompt-Level Only — Not Machine-Enforced

- **What:** The design's core value proposition — typed schemas at every boundary, article principle #1 compliance — rests on schema validation. But the design explicitly concedes: "Validation is prompt-level (the orchestrator is instructed to check required fields), not runtime-enforced. This is a pragmatic concession to the VS Code agent framework's capabilities" (§Decision 4). This means the "schema validation" layer is another LLM reading text and checking if fields appear present. An LLM-based validator has the same failure modes as the agents producing the output (hallucination, context drift, overlooking missing fields). There is no actual YAML parser, no JSON Schema validator, no enforcement layer between agents.
- **Where:** [design.md](../design.md) §Decision 4 (Schema Validation Protocol); affects all downstream claims about typed communication (AC-8), deterministic routing (AC-9), and evidence gating (AC-5)
- **Likelihood:** High — prompt-level validation is inherently probabilistic
- **Impact:** High — the entire typed-YAML architecture is justified by machine-parseability, but the "machine" parsing it is an LLM. The design pays the YAML complexity costs (indentation sensitivity, LLM formatting errors requiring retries, special character escaping) without genuine machine-parsing benefits. An LLM can extract `status: DONE` from YAML or from `## Status: DONE` in Markdown with comparable reliability.
- **Assumption at risk:** That typed YAML provides materially stronger validation guarantees than structured Markdown sections when both are validated by an LLM reading text. The article's principle #1 assumes a real enforcement layer exists.

---

### [Severity: High] "Zero-Merge" Redistributes Cost Rather Than Eliminating It

- **What:** The design claims zero memory merge operations, eliminating Forge's ~12 merges per run. While technically true (no explicit merge step exists), the work of synthesizing multi-source information is redistributed to every downstream agent. Each downstream agent must now read the pipeline manifest + N upstream YAML outputs and mentally synthesize them, where Forge agents read one pre-merged `memory.md`. For agents late in the pipeline — Verifier (reads implementation reports + plan + spec + design), Adversarial Reviewer (reads code diff + implementation reports + verification evidence + spec), Knowledge Agent (reads ALL outputs) — this means reading 10+ separate files. The context window pressure may be WORSE than Forge's single merged summary.
- **Where:** [design.md](../design.md) §Decision 5 (Memory System Redesign); [design.md](../design.md) §Decision 3 (Pipeline Structure, Steps 6-8 inputs)
- **Likelihood:** Medium — depends on feature complexity and agent context window sizes
- **Impact:** High — if downstream agents hit context limits from reading many upstream files, the pipeline fails at precisely the stages (verification, review) where failure is most costly. Forge's merge operation, while overhead, was a context-efficiency optimization.
- **Assumption at risk:** That reading N typed YAML files is cheaper (context-wise) than reading one curated merged summary. YAML headers, schema boilerplate, and repeated structural elements across files may consume more tokens than equivalent Markdown prose.

---

### [Severity: High] SQL Evidence Gating Inaccessible to Orchestrator

- **What:** The evidence gating mechanism defines SQL COUNT queries for gate evaluation (e.g., `SELECT COUNT(*) FROM verification_ledger WHERE task_id=? AND phase='baseline' > 0`). But the orchestrator cannot execute SQL queries — `run_in_terminal` is in its forbidden tools list. This means either: (a) the orchestrator relies on the Verifier's self-reported counts in its typed YAML output (undermining the anti-hallucination property of SQL gating), or (b) the orchestrator reads the YAML fallback file and counts entries manually (making SQL the "primary" backend irrelevant for gating decisions), or (c) there's an undocumented mechanism.
- **Where:** [design.md](../design.md) §Decision 6 (Evidence Gating Mechanism table); §Decision 2 (Orchestrator tool restrictions)
- **Likelihood:** High — this is a structural tool-access conflict
- **Impact:** High — the SQL ledger's core benefit is "INSERT-before-report eliminates hallucinated verification" (from research findings). If the orchestrator can't independently verify SQL counts and must trust the Verifier's output, the anti-hallucination property is weakened. The evidence gate becomes: trust what the Verifier says its counts are.
- **Assumption at risk:** That the orchestrator can independently verify evidence counts against the SQL ledger. It cannot, given its tool restrictions.

---

### [Severity: High] Custom Agent Model Routing May Not Work

- **What:** Anvil's multi-model adversarial review uses `agent_type: "code-review"` — a platform-provided VS Code Copilot agent type that supports model routing. The new design replaces this with a custom `adversarial-reviewer.agent.md` dispatched via `runSubagent` with a `model` parameter. Whether `runSubagent` supports model routing for custom `.agent.md` agents (as opposed to built-in `code-review` agents) is unverified. The research explicitly flags this: "Can VS Code's `runSubagent` mechanism specify the LLM model for a subagent invocation?" (research/dependencies.md §Open Questions). The design acknowledges the risk (EC-9 fallback) but treats multi-model dispatch as the primary path.
- **Where:** [design.md](../design.md) §Decision 7 (Adversarial Review Design, Dispatch parameters); [design.md](../design.md) §Decision 2 (Agent 8: Adversarial Reviewer dispatch parameters)
- **Likelihood:** Medium — platform capability is genuinely unknown; the fallback exists but degrades quality
- **Impact:** High — if model routing doesn't work for custom agents, the core differentiator of adversarial review (genuinely different LLM perspectives) disappears. Same-model-different-prompt fallback is essentially what Forge's CT cluster already did — the very pattern the design replaces as inferior.
- **Assumption at risk:** A-2: "The VS Code Copilot runtime supports dispatching `code-review` subagents to specific LLM models." The design extends this assumption to custom agents without evidence.

---

### [Severity: High] The Synthesis Is Direction A With Bolt-Ons, Not a Genuine Hybrid

- **What:** The design claims a "Synthesized A+D+B" architecture, but the actual elements taken from each direction reveal it is substantially Direction A with additions: Direction A's lean agent count (6-8 → 9), Direction A's zero-merge memory, Direction A's pipeline structure. From D: schema rigor (an additive requirement applicable to any direction) and SQL dual-track (a verification backend choice). From B: Knowledge Agent retention (one agent). Calling this "A+D+B" overstates the synthesis — it's Direction A enhanced, not a three-way hybrid. This matters because the design scores pure Direction A at 7.5/10 and the synthesis higher, but the score difference should be minimal since the actual architectural decisions are almost entirely A.
- **Where:** [design.md](../design.md) §Decision 1 (Selected Approach: Synthesized A+D+B, element table)
- **Likelihood:** Low — this is a framing issue, not a functional risk
- **Impact:** Medium — overstating synthesis may cause implementers to miss that this is an opinionated lean-pipeline design. More importantly, it obscures whether Direction D's full schema rigor (which scored 8/10) would have been a better pure choice, since D already includes typed schemas AND a reasonable agent count (8-10).
- **Assumption at risk:** That synthesizing elements from 3 directions produces something better than any pure direction. The synthesis may have inherited A's weaknesses (minimal schema rigor) while only partially adopting D's strengths.

---

### [Severity: Medium] Always-On Adversarial Review Drops Anvil's Task-Sizing Optimization

- **What:** Anvil defines three task sizes: Small (skips adversarial review entirely), Medium (1 reviewer), Large (3 reviewers). The new design drops the Small category — every pipeline run gets both adversarial design review AND adversarial code review, even for trivial changes (adding a config file, updating docs). For a 🟢-only feature adding test files, the pipeline still dispatches at least 2 adversarial review instances. Anvil's optimization recognized that some changes don't warrant review overhead.
- **Where:** [design.md](../design.md) §Decision 7 (Trigger Conditions: "Adversarial review is always triggered"); contrast with Anvil's task-sizing model ([Anvil/anvil.agent.md](../../Anvil/anvil.agent.md) §Task Sizing)
- **Likelihood:** Medium — depends on how often trivial features are processed
- **Impact:** Medium — 2 unnecessary dispatches per trivial feature add latency and cost. The design argues "overhead of 1 reviewer per standard feature is minimal compared to eliminated CT/R dispatches," but this compares against the wrong baseline — Anvil skipped review entirely for Small tasks.
- **Assumption at risk:** That consistent review quality justifies always-on review. For genuinely trivial changes, the review overhead exceeds the risk of skipping it.

---

### [Severity: Medium] Pushback in Autonomous Mode Is a No-Op for Quality Assurance

- **What:** In autonomous mode, the pushback system "logs concerns and auto-proceeds." The Spec agent evaluates request quality, potentially identifies requirements conflicts or dangerous edge cases (the "expensive kind" from Anvil), logs them, and proceeds anyway. Pushback was designed as a quality gate — a mechanism to prevent bad requests from consuming pipeline resources. In autonomous mode, it provides audit trail value but zero quality-gate value. The design doesn't distinguish between severity levels of pushback concerns — a "this will corrupt user data" concern is logged-and-ignored the same as a "there's a simpler approach" concern.
- **Where:** [design.md](../design.md) §Decision 9 (Mode Switching Behavior); CR-14 specification in [feature.md](../feature.md)
- **Likelihood:** Medium — autonomous pipelines processing risky requests will encounter this
- **Impact:** Medium — the pipeline may waste significant compute on a flawed request that pushback identified, only to fail at verification. Severity-based autonomous pushback (block on Blocker/Critical concerns even in autonomous mode) would preserve some quality-gate value.
- **Assumption at risk:** That logging pushback concerns is sufficient quality assurance in autonomous mode. It's sufficient for audit trails but not for preventing wasted execution on known-bad requests.

---

### [Severity: Medium] FR-4.7 Revert Mechanism Is Unassigned

- **What:** FR-4.7 requires: "After 2 failed fix attempts for a verification check, the system MUST revert changes for that check and record the failure." The design mentions this requirement in passing ("After 2 failed fix attempts for a specific verification check: revert changes for that check, record the failure in the ledger") but does not specify which agent performs the revert. The Implementer could revert (but may have exited). The Verifier can run git commands but MUST NOT modify source files. The Orchestrator can't run terminal commands. This requirement is acknowledged but architecturally unresolved.
- **Where:** [design.md](../design.md) §Decision 6 (Verification-Replan Loop, after the iteration diagram); FR-4.7 in [feature.md](../feature.md)
- **Likelihood:** Medium — reverts are needed when implementation fails verification twice
- **Impact:** Medium — without a clear owner for the revert operation, implementations may proceed with known-broken changes rather than reverting, undermining the verification guarantee
- **Assumption at risk:** That the revert mechanism can be resolved at implementation time. The architectural tool-access constraints make this non-trivial — the most natural agent for reverts (Verifier, which already detects failures) is prohibited from modifying source files.

---

### [Severity: Medium] Orchestrator Complexity Is Likely to Exceed Context Window

- **What:** The new orchestrator has MORE responsibilities than Forge's 544-line orchestrator: schema validation for every agent output, evidence gate evaluation, risk assessment for reviewer count decisions, pipeline manifest management, SQL availability detection, approval mode handling, and the full routing table. Yet the design doesn't address orchestrator prompt size or context window pressure. If the orchestrator prompt exceeds practical context limits, it becomes the single point of failure for the entire pipeline — and the most complex agent in the system, contradicting the "lean" design philosophy.
- **Where:** [design.md](../design.md) §Decision 2 (Orchestrator agent detail); §Decision 4 (Schema Validation Protocol); §Decision 6 (Evidence Gating); §Decision 9 (Approval Mode)
- **Likelihood:** Medium — Forge's 544-line orchestrator already "pushes context limits" per research/impact.md
- **Impact:** High — orchestrator failure cascades to total pipeline failure. Context-window-induced drift in the orchestrator could cause it to skip validation steps, misroute agents, or bypass evidence gates.
- **Assumption at risk:** That the orchestrator can handle schema validation, evidence gating, risk assessment, manifest management, and routing logic within a single agent definition's context budget. The existing orchestrator at 544 lines already strains limits without these additional responsibilities.

---

### [Severity: Low] Dispatch Count Claim Is Misleading Without Manifest Write Overhead

- **What:** The design claims "18-22 dispatches vs. Forge's ~32+" for a typical 6-task feature. This count excludes: (a) manifest update dispatches (the orchestrator can't write files, so ~8+ dispatches for manifest updates), (b) Step 0 setup dispatches (initializing manifest, checking prerequisites), (c) any revision/replan loop dispatches. A more honest count for the happy-path without revisions but including manifest writes would be ~26-30, which is only ~10% better than Forge, not the ~40% improvement implied.
- **Where:** [design.md](../design.md) §Decision 3 (Dispatch Count Analysis table)
- **Likelihood:** High — the manifest write problem is structural (see Finding 1)
- **Impact:** Low — the dispatch count reduction is a secondary benefit, not the core value proposition. The real improvements (typed schemas, evidence gating, reduced agent count) still hold.
- **Assumption at risk:** That pipeline state management is free from a dispatch perspective. It isn't, given the orchestrator's tool restrictions.

---

## Cross-Cutting Observations

- **Security (ct-security scope):** The design's model-routing fallback (EC-9) means same-model-different-prompt for security review — which is precisely Forge's CT pattern that the design calls insufficient. If model routing fails, the security review quality regresses to Forge-level, but the system reports it as "adversarial review complete" with a warning. The warning may not be surfaced prominently enough.

- **Maintainability (ct-maintainability scope):** The 14 YAML schemas defined in `schemas.md` create a tight coupling between the schema reference document and every agent definition. Schema evolution requires synchronized updates to `schemas.md` plus all agents that produce/consume that schema. The design doesn't address schema versioning beyond `schema_version: "1.0"`.

- **Scalability (ct-scalability scope):** The zero-merge model requires each downstream agent to read an increasing number of upstream files. For complex features with many tasks, the Knowledge Agent at Step 8 must read ALL pipeline outputs — potentially dozens of YAML files. This is an O(N) scaling concern that Forge's merge model addressed (albeit with overhead).

---

## Requirement Coverage

| Requirement                   | Coverage Status      | Notes                                                                                                |
| ----------------------------- | -------------------- | ---------------------------------------------------------------------------------------------------- |
| CR-1: Deterministic Pipeline  | ✅ Fully covered     | Pipeline is deterministic; routing table is explicit                                                 |
| CR-2: Three-State Contracts   | ✅ Fully covered     | All 9 agents have completion contracts                                                               |
| CR-3: Parallel Dispatch       | ✅ Fully covered     | Pattern A with ≤4 concurrency cap, sub-wave partitioning                                             |
| CR-4: Adversarial Multi-Model | ⚠️ At risk           | Depends on A-2 (model routing for custom agents); fallback degrades to Forge-level same-model review |
| CR-5: Evidence Gating         | ⚠️ Partially covered | SQL gates defined but orchestrator cannot execute SQL queries; prompt-level validation only          |
| CR-6: Risk Classification     | ✅ Fully covered     | Per-file 🟢/🟡/🔴; drives verification depth and reviewer count                                      |
| CR-7: Justification Scoring   | ✅ Fully covered     | Decision record schema with confidence levels                                                        |
| CR-8: Approval Mode           | ✅ Fully covered     | Structured multiple-choice at 2 gates; autonomous auto-proceed                                       |
| CR-9: No Pending Step         | ✅ Fully covered     | All steps execute or are skipped deterministically                                                   |
| CR-10: Output Directory       | ✅ Fully covered     | `NewAgents/.github/` for agents, `NewAgents/` for README                                             |
| CR-11: Self-Verification      | ✅ Fully covered     | Agent template includes self-verification section                                                    |
| CR-12: Anti-Drift Anchors     | ✅ Fully covered     | Agent template includes anti-drift anchor                                                            |
| CR-13: Unified Severity       | ✅ Fully covered     | Blocker/Critical/Major/Minor; severity-taxonomy.md reference                                         |
| CR-14: Pushback System        | ⚠️ Partially covered | Functional in interactive mode; no-op for quality assurance in autonomous mode                       |
| CR-15: Bounded Retries        | ✅ Fully covered     | All loops have explicit maximums; worst-case bounded                                                 |
| FR-4.7: Revert on 2 failures  | ⚠️ Partially covered | Mentioned but no agent assigned for revert operation                                                 |
| FR-6.2: Memory-first reading  | ✅ Covered           | Preserved via pipeline manifest → output summaries → targeted reads                                  |
| NFR-3: Scalability            | ⚠️ At risk           | Zero-merge model may cause context pressure for late-pipeline agents on complex features             |
| NFR-5: Context Efficiency     | ⚠️ At risk           | Orchestrator accumulates many responsibilities; downstream agents read many files                    |
