# Adversarial Design Review — Reviewer 3 (claude-opus-4.6 perspective)

## Overall Verdict: CONCERNS

**Summary:** The design is thorough, well-documented, and honestly self-aware about its limitations. However, it has a fundamental identity crisis: it claims to merge Forge and Anvil's best qualities, but in practice it inherited Forge's pipeline structure and lost Anvil's most powerful patterns. The complexity didn't decrease — it migrated from inter-agent orchestration to intra-agent prompt complexity. Several core value propositions (multi-model review, typed schema enforcement, evidence bundles) rest on unverified platform assumptions or prompt-level-only mechanisms that the design itself admits cannot guarantee correctness. The tri-format storage approach is a complexity tax that Anvil solved more elegantly with two formats.

---

## Findings

### [Finding 1]: Multi-Model Routing for Custom Agents Is the Design's Load-Bearing Assumption — And It's Unverified

- **Severity:** Critical
- **Area:** §Decision 7 (Adversarial Review Design), §Decision 2 (Adversarial Reviewer agent detail), Assumption A-2
- **Issue:** The entire adversarial review architecture — the design's primary differentiator over Forge's CT cluster — depends on dispatching a _custom_ `adversarial-reviewer.agent.md` to three _different_ LLM models via `runSubagent`. But Anvil achieves multi-model review using `agent_type: "code-review"` — a **platform-provided** VS Code Copilot agent type, not a custom `.agent.md` file. The research explicitly flags this as an open question: "Can VS Code's `runSubagent` mechanism specify the LLM model for a subagent invocation?" (research/dependencies.md §Open Questions). There is zero evidence that `runSubagent` supports model routing for custom agent definitions. The design acknowledges this (EC-9 fallback) but treats multi-model dispatch as the primary path and builds 6 dispatches per pipeline run around it.
- **Impact:** If model routing doesn't work for custom agents, the fallback is same-model-different-prompt — which is _precisely_ what Forge's CT cluster already does. The design explicitly criticizes this pattern as inferior ("~50% shared prompt content with siblings", "3-iteration bottleneck, 12 dispatches"). The fallback path negates the design's core improvement over Forge. Worse, the design would dispatch 3 instances of the same model with the same prompt (since the adversarial reviewer uses a single definition), making the fallback _less_ diverse than Forge's 4 CT agents which at least had different focus areas (security, scalability, maintainability, strategy).
- **Suggested fix:** Either (a) keep Anvil's actual pattern: use the platform `code-review` agent type with model parameters for adversarial review, accepting it's a built-in agent, not a custom one; or (b) design the adversarial reviewer as 3 separate agent definitions with genuinely different review focus areas (security-focused, correctness-focused, architecture-focused), providing perspective diversity even when model diversity is unavailable; or (c) verify model routing capability before committing the architecture — if it's unverifiable, design the primary path around same-model-different-prompt and treat multi-model as the enhancement. Don't build the primary architecture around an unverified assumption and hand-wave the fallback.

---

### [Finding 2]: Complexity Was Relocated, Not Reduced — The Verifier and Adversarial Reviewer Are Mega-Agents

- **Severity:** High
- **Area:** §Decision 2 (Agent Inventory), §Decision 6 (Verification Architecture), §Decision 7 (Adversarial Review Design)
- **Issue:** The design claims to reduce from 23 agents to 9 "unique definitions." But this is sleight of hand. The Verifier alone absorbs 4 Forge agents (V-Build, V-Tests, V-Tasks, V-Feature). The Adversarial Reviewer absorbs 7 (4 CT + 3 R-agents). That's 11 agents → 2. The work those 11 agents performed didn't vanish — it was compressed into two complex mega-prompts:
  - **Verifier** must: execute a 3-tier verification cascade, manage the SQLite `anvil_checks` ledger (CREATE TABLE, INSERT, SELECT), perform baseline cross-checks, detect regressions by comparing baseline vs. after states, verify acceptance criteria against the spec, dynamically discover build/test tooling, and handle tier-3 infeasibility gracefully. The design itself calls this "the most complex agent."
  - **Adversarial Reviewer** must handle two fundamentally different tasks (design review examining architecture decisions vs. code review examining `git diff --staged`) via parameterization, creating a 2×3×3 = 18 configuration space. Design review needs architectural reasoning; code review needs bug-hunting. A single prompt general enough for both will be mediocre at each.

  The prior ct-maintainability review already flagged this: "The Verifier is essentially an internal state machine living inside a single prompt. This is the definition of a mega-agent." The v3 response (per-task dispatch) bounds context but doesn't reduce prompt complexity.

- **Impact:** The design pays the complexity of 23 agents in agent prompt size rather than in orchestration overhead. This is a tradeoff, not a simplification. The Verifier prompt will likely exceed 400+ lines (Anvil's entire agent is 419 lines and does less). Context window pressure on the Verifier and Adversarial Reviewer becomes the new bottleneck, replacing Forge's dispatch coordination bottleneck.
- **Suggested fix:** Acknowledge this honestly as a tradeoff rather than claiming "reduced complexity." Consider splitting the Verifier into two: a Build/Test Verifier (Tiers 1-2, tool execution, SQL recording — genuinely bounded scope) and an Acceptance Verifier (Tier 3 + acceptance criteria + regression detection — reasoning scope). This adds 1 agent (10 total, still within range) but creates a natural separation between mechanical verification and reasoning-based verification.

---

### [Finding 3]: Anvil's Tight Verify-Fix Loop Was Lost in Translation — The Most Powerful Pattern Is Gone

- **Severity:** High
- **Area:** §Decision 3 (Pipeline Structure, Steps 5-6), §Decision 6 (Verification-Replan Loop)
- **Issue:** Anvil's single most powerful pattern is its tight inner loop: the SAME agent that writes code (Step 4) immediately runs verification (Step 5b), and if a check fails, fixes it on the spot (max 2 attempts), and if it still can't fix it, reverts. Crucially, the fixing agent has full context — it just wrote the code, it just saw the error, it knows exactly what went wrong. The feedback loop is milliseconds-fast and context-complete.

  The new design replaces this with a cross-agent round trip: Implementer writes code → dispatched back to orchestrator → Verifier dispatched to check → reports NEEDS_REVISION → orchestrator routes to Planner → Planner replans → new Implementer dispatched. That's 4+ dispatch boundaries with context loss at each one. The new Implementer that gets the replanned task has never seen the original code it's fixing. The verification finding is mediated through a Planner that may not understand the technical nuance.

  This is the highest-value pattern Anvil has, and the multi-agent pipeline structurally cannot replicate it. Forge had the same limitation — and Anvil was built specifically to solve it.

- **Impact:** Verification failures that Anvil fixes in one tight loop become multi-dispatch, multi-agent ordeals. A simple "forgot to import X" that Anvil catches and fixes in 10 seconds becomes: Verifier dispatch → failure report → Planner replan → Implementer re-dispatch, consuming 3-4 dispatches and potentially minutes. For a 6-task feature with even one failing verification per task, the difference is ~6 extra dispatches vs. zero.
- **Suggested fix:** Give the Implementer a self-verification step that mirrors Anvil's structure: after implementing, run IDE diagnostics + build + tests. If anything fails, fix immediately (max 2 attempts within the Implementer's context). Only dispatch the Verifier for _independent_ verification after the Implementer's self-fix loop passes. This preserves Anvil's tight loop for simple fixes while keeping the independent Verifier for cross-checking. The design briefly mentions "basic self-check" in Step 5 but doesn't specify what this means or give it Anvil's fix-and-revert teeth.

---

### [Finding 4]: Anvil's Evidence Bundle — The Key User-Facing Deliverable — Was Dropped

- **Severity:** High
- **Area:** §Decision 6 (Verification Architecture), §Decision 3 (Pipeline Structure), compared to Anvil §5e
- **Issue:** Anvil's Step 5e generates a structured Evidence Bundle — a carefully formatted human-readable summary that includes: baseline state (before changes), verification results (after changes), regressions detected, adversarial review verdicts per model, issues fixed before presenting, changes summary, blast radius, confidence level with precise definitions, and rollback command. This is the primary deliverable the user sees. It's what makes Anvil _trustworthy_ — the user doesn't have to review the code to know it was verified, because the evidence is structured and auditable.

  The new design has no equivalent. The Verifier writes `verification-reports/<task-id>.yaml`. The Adversarial Reviewer writes `review-findings/<scope>-<model>.md`. The Knowledge Agent writes `knowledge-output.yaml`. But no agent assembles these into a coherent, user-facing evidence bundle. There's no confidence rating applied to the overall pipeline output. There's no rollback command. There's no "issues fixed before presenting." There's no blast radius analysis. The user gets... a collection of files?

- **Impact:** The user experience degrades from "here's a single structured proof of quality" to "go read 6+ files across 3 formats to assess confidence." Anvil's Evidence Bundle was the user-facing manifestation of evidence-first engineering. Dropping it drops the entire user trust story.
- **Suggested fix:** Add an Evidence Bundle assembly step. This could be the Knowledge Agent's responsibility (it already reads all pipeline outputs at Step 8) or a final orchestrator synthesis step. The bundle should include: (a) overall confidence rating with Anvil's precise definitions (High/Medium/Low), (b) aggregated verification summary (passed/failed/regressions), (c) adversarial review summary (verdicts per model, issues fixed), (d) rollback command, (e) blast radius. Present this to the user as the pipeline's final output.

---

### [Finding 5]: The SQLite + YAML + Markdown Tri-Format Storage Is a Complexity Tax That Anvil Didn't Need

- **Severity:** High
- **Area:** §Data Storage Strategy (v3), §Decision 4 (Communication & State), §Decision 6 (Verification Architecture)
- **Issue:** The design uses three data formats:
  - **SQLite** for verification evidence (`anvil_checks`) and telemetry (`pipeline_telemetry`)
  - **YAML** for all agent I/O schemas (typed outputs, completion contracts)
  - **Markdown** for human-readable companions, review findings, agent definitions, schema documentation

  Every agent that does verification work must be literate in at least two formats: the Verifier writes SQL AND YAML, the Adversarial Reviewer writes Markdown AND SQL, the Orchestrator reads YAML AND queries SQL. The §Data Storage Strategy section admits "No format serves all needs" — but Anvil solved this with just two formats (SQL + Markdown) and didn't need a third.

  The YAML layer is the weakest link. The design itself admits: validation is "prompt-level structural guidance, not machine enforcement" (§Decision 4). YAML adds indentation sensitivity, requires escaping for special characters, and LLMs frequently produce malformed YAML. The design's mitigation is "retry on malformed output" — adding dispatch overhead for a format choice. If YAML provides no runtime guarantees over structured Markdown (which the design concedes), why add it? The "typed schema" value proposition requires enforcement that doesn't exist.

- **Impact:** (a) Agent prompt complexity increases — each agent must know its format(s). (b) Format boundary bugs become a new failure class — what happens when the YAML `gate_status` disagrees with the SQL COUNT? The design says "SQL wins" but the orchestrator reads YAML first for routing. (c) Schema evolution now spans three formats. (d) Debugging requires reading SQL + YAML + Markdown for a single pipeline step.
- **Suggested fix:** Reduce to two formats: **SQLite** (all structured/queryable data: verification, telemetry, agent completion signals, decision records) and **Markdown** (all human-readable: agent definitions, review findings, companion docs). This matches Anvil's actual pattern. Agent completion signals go in a `pipeline_state` SQL table instead of YAML files. The orchestrator queries SQL for both routing AND evidence gating — one query mechanism, not two. YAML is eliminated as a format; its "typed schema" benefits can be achieved with structured SQL tables that actually ARE machine-enforceable (via CHECK constraints, NOT NULL, etc.).

---

### [Finding 6]: Pushback Fires After 4 Researchers Run — Anvil's "Evaluate Before Executing" Is Structurally Impossible Here

- **Severity:** High
- **Area:** §Decision 9 (Approval Mode), §Decision 2 (Spec agent), compared to Anvil §Pushback
- **Issue:** Anvil's pushback fires at Step 0, _before any work happens_. It's the first thing Anvil does: "Before executing any request, evaluate whether it's a good idea — at both the implementation AND requirements level. If you see a problem, say so and stop for confirmation." This means Anvil spends zero compute on bad requests.

  In the new pipeline, pushback lives in the Spec agent at Step 2. By the time it fires, 4 researchers have already been dispatched and completed (Step 1). That's 4 dispatches of wasted compute — the most expensive parallel step in the pipeline — on a request that pushback might reject.

  Worse, in autonomous mode, pushback "logs concerns and auto-proceeds" (§Decision 9). This means pushback in autonomous mode is a no-op for quality assurance — it provides audit trail but doesn't prevent wasted execution on known-bad requests.

- **Impact:** For flawed requests in autonomous mode, the pipeline burns through all 26+ dispatches before producing output that pushback would have caught at Step 0. The "evaluate before executing" philosophy that makes Anvil efficient is architecturally impossible in a pipeline where research precedes evaluation.
- **Suggested fix:** (a) Move pushback to Step 0 or embed it in the orchestrator's Setup phase, before researchers are dispatched. The orchestrator can evaluate the initial request for obvious pushback triggers (scope too large, conflicting requirements, vague specifications) without needing research. (b) For autonomous mode, add severity-based pushback blocking: if pushback identifies a Blocker-severity concern (e.g., "this will delete user data without confirmation"), halt even in autonomous mode. Log-and-proceed for Major/Minor concerns.

---

### [Finding 7]: Justification Scoring Is Self-Assessment, Not Scoring

- **Severity:** Medium
- **Area:** §Decision 8 (Justification Scoring Mechanism), CR-7
- **Issue:** The design presents a `decision-record.schema` with `confidence: High | Medium | Low` and definitions for each level. But the "scoring" is entirely self-assessed by the producing agent (Designer or Planner). The Designer decides its own confidence is "High" because "all alternatives were evaluated with clear evidence." There's no:
  - Minimum number of alternatives required (an agent could list 1 alternative and call it "all")
  - Scoring rubric with criteria that must be satisfied
  - Independent validation (no downstream agent challenges confidence ratings)
  - Calibration mechanism (how do we know the Designer's "High" means the same as the Planner's "High"?)
  - Consequence for low confidence (a "Low" confidence decision proceeds through the pipeline identically to "High")

  The feature spec (CR-7) requires "justification scores" — the design delivers "justification self-labels." These are not the same thing.

- **Impact:** Justification scoring becomes documentation theater. Every decision gets labeled, but the labels carry no actionable weight. A series of "Low confidence" design decisions won't trigger additional review or escalation — they'll proceed to planning and implementation just like "High confidence" decisions.
- **Suggested fix:** (a) Make the Adversarial Design Review explicitly evaluate justification quality: reviewers should challenge decisions where the confidence self-assessment seems unsupported by the listed alternatives and rationale. (b) Add a minimum requirement: decisions must consider ≥2 alternatives with pros/cons. (c) Low confidence decisions should trigger escalation: route to interactive approval even in autonomous mode, or require the adversarial review to specifically address those decisions.

---

### [Finding 8]: Anvil's Git Hygiene, Session Recall, and Auto-Commit Were Silently Dropped

- **Severity:** Medium
- **Area:** §Decision 2 (Agent Inventory), §Decision 3 (Pipeline Structure), compared to Anvil §0b, §1b, §8
- **Issue:** Three Anvil features with no equivalent in the new design:

  **Git Hygiene (Anvil Step 0b):** Before any work, Anvil checks `git status --porcelain` for dirty state, verifies you're not on `main` for Medium/Large tasks, and detects worktrees. This prevents mixing uncommitted changes from previous tasks with new work. The new pipeline has no equivalent — the Setup step (Step 0) checks for "git, build tools, VS Code tools" but doesn't check working tree state.

  **Session Recall (Anvil Step 1b):** Anvil queries the `session_store` SQL database for past sessions that touched the same files, looking for previous regressions or established patterns. This prevents repeating past mistakes. The new pipeline has VS Code `store_memory` for cross-session knowledge but no systematic recall mechanism that checks "did we break this file before?"

  **Auto-Commit (Anvil Step 8):** After presenting results, Anvil automatically commits with a structured message and `Co-authored-by` trailer, providing the rollback SHA. The new pipeline ends at Knowledge Capture (Step 8) with no commit step. The user must manually commit all changes.

- **Impact:** (a) Git hygiene: users start the pipeline with dirty state, mixing old uncommitted work with new changes, making rollback impossible. (b) Recall: the pipeline repeats past mistakes on files that previously caused regressions. (c) Auto-commit: work isn't preserved until the user manually commits, creating a window where VS Code crashes or power loss could lose all pipeline output.
- **Suggested fix:** (a) Add git hygiene checks to Step 0 (Setup). Orchestrator can run `git status --porcelain` and `git rev-parse --abbrev-ref HEAD` via `run_in_terminal` and present pushback if state is dirty. (b) Add recall as an optional Implementer step: before implementing, query `store_memory` for past issues with the target files. (c) Add auto-commit as Step 8b or as the final Implementer action: `git add -A && git commit -m "..."`.

---

### [Finding 9]: The Orchestrator's `run_in_terminal` Access Reverses a Prior Deliberate Restriction

- **Severity:** Medium
- **Area:** §Decision 2 (Orchestrator tools), §v3 Revision Log (Correction #1)
- **Issue:** The `orchestrator-tool-restriction` feature — an entire design/implementation cycle in this workspace — explicitly prohibited the Forge orchestrator from using `run_in_terminal`. The verification artifacts confirm this was deliberate and thorough: "The orchestrator MUST NOT use `create_file`, `replace_string_in_file`, `multi_replace_string_in_file`, `run_in_terminal`, `grep_search`, `semantic_search`, `file_search`, or `get_errors`." The rationale was isolation: the orchestrator should coordinate via dispatch, not perform actions directly.

  The v3 design reverses this, giving the orchestrator `run_in_terminal` and `get_terminal_output` explicitly for SQLite queries. The v3 Revision Log justifies this as "SQLite first-class" but doesn't acknowledge:
  - This contradicts a deliberate prior architectural decision
  - The original restriction rationale (preventing the orchestrator from side-effecting) still applies to other uses of `run_in_terminal`
  - Once `run_in_terminal` is allowed, there's no mechanism to restrict it to _only_ SQL queries — the orchestrator could run arbitrary commands
  - The design's own anti-drift anchors would need to address this expanded attack surface

- **Impact:** The orchestrator gains the ability to run arbitrary terminal commands, not just SQL queries. A drifted orchestrator could modify files via `echo "..." > file.txt` in the terminal, bypassing the file-write restriction. The isolation model is weakened.
- **Suggested fix:** (a) Acknowledge the philosophical reversal explicitly and justify why the new tradeoff is acceptable. (b) Constrain the orchestrator's `run_in_terminal` usage in the anti-drift anchor: "You use `run_in_terminal` ONLY for SQLite queries on the verification ledger. You NEVER use it for file operations, git commands, build commands, or any other purpose." (c) Alternatively, keep the orchestrator read-only and have the Verifier expose a summary SQL view that the orchestrator reads as a file — preserving the original isolation model.

---

### [Finding 10]: Always-3 Reviewers for Every Feature Wastes 6 Dispatches on Trivial Changes

- **Severity:** Medium
- **Area:** §Decision 7 (Trigger Conditions), §v3 Revision Log (Correction #2)
- **Issue:** Anvil's task sizing is one of its smartest features: Small tasks (typo, rename, config tweak) get zero adversarial review. Medium tasks get 1 reviewer. Only Large tasks get 3 reviewers. This is a deliberate cost-benefit optimization — the review overhead should be proportional to the risk.

  The v3 design dispaches 3 reviewers for BOTH design review AND code review — always, regardless of risk. For a pure-🟢 feature adding documentation files: 3 design reviewers + 3 code reviewers = 6 reviewer dispatches. Anvil would dispatch zero. The v3 justification is "simplification" (removes conditional logic from orchestrator) and "consistent quality." But:
  - "Simplification" at the orchestrator level adds cost at the execution level
  - "Consistent quality" for a docs-only change is overkill — there is no security, correctness, or architectural risk
  - The condition removed (`if risk >= 🔴: 3 reviewers; else: 1`) is a 2-line if/else — the "simplification" saves negligible orchestrator complexity

  Furthermore, the design dropped Anvil's "Small" task category entirely. There's no way to express "this change is too trivial for review."

- **Impact:** For a pipeline that runs frequently on minor changes, 6 wasted dispatches per run adds up. At ~2-3 min per dispatch, that's 12-18 minutes of unnecessary review. The feature spec (CR-4) actually specifies risk-driven reviewer count ("1 for standard, 3 for Large/🔴") — the v3 change diverges from the spec.
- **Suggested fix:** (a) Restore risk-driven reviewer count as the spec requires: 1 reviewer for standard, 3 for Large/🔴. The orchestrator condition is trivial. (b) Consider restoring Anvil's Small task path: for pure-🟢 features with no business logic changes, skip adversarial review entirely at the Planner's recommendation (recorded in `plan-output.yaml`). (c) If always-3 is retained, explicitly acknowledge the spec divergence from CR-4 and FR-5.3.

---

### [Finding 11]: Forge's Specialized Review Perspectives Were Lost — Model Diversity ≠ Perspective Diversity

- **Severity:** Medium
- **Area:** §Decision 2 (Agents Removed table), §Decision 7 (Adversarial Review Design)
- **Issue:** Forge's CT cluster had 4 agents with genuinely different review _perspectives_:
  - **ct-security:** Focused on authentication gaps, data exposure, injection vectors
  - **ct-scalability:** Focused on single points of failure, unbounded growth, missing pagination
  - **ct-maintainability:** Focused on tight coupling, missing abstractions, unclear boundaries
  - **ct-strategy:** Focused on overall approach correctness, scope, long-term implications

  The design replaces all four with a single `adversarial-reviewer.agent.md` dispatched to 3 different models. But model diversity is not the same as perspective diversity. Three different LLMs given the same prompt will produce similar feedback because they're answering the same question. Forge's CT agents produced _different kinds of feedback_ because they were asking different questions. The design criticizes CT for "~50% shared prompt content" — but the 50% that was different is what provided specialized perspectives.

  The R-cluster loss is similar: r-quality, r-security, and r-testing each had focused review criteria. The new adversarial reviewer must cover all these concerns in a single prompt.

- **Impact:** Review quality becomes broad but shallow. Three models finding the same 5 obvious issues vs. four specialized agents each finding domain-specific issues they're prompted to look for. The design may catch more correlated bugs (all three models notice a null pointer) but fewer domain-specific bugs (only a security-focused reviewer would probe for SSRF in a URL input).
- **Suggested fix:** If keeping a single agent definition, add perspective-specific prompting: when dispatching the 3 reviewers, vary the prompt to emphasize different concerns (security + injection vectors for model 1, architecture + coupling for model 2, correctness + edge cases for model 3). This combines model diversity with the perspective diversity that made CT effective. The design already has a `review_scope` parameter — extend it with a `review_focus` parameter.

---

### [Finding 12]: "Prompt-Level Structural Guidance" Is Honest — But Then Why Build an Architecture Around It?

- **Severity:** Medium
- **Area:** §Decision 4 (Schema Validation Protocol), §CT Revision Log Finding #2
- **Issue:** The design deserves credit for its honesty: the v2 CT revision explicitly recharacterizes schema validation as "prompt-level structural guidance, not machine enforcement" and states "there is no runtime YAML parser, no JSON Schema validator, no enforcement layer between agents." This is refreshingly honest.

  But the honesty creates a philosophical contradiction: if typed YAML schemas provide no runtime guarantees over structured Markdown — and the design admits this — then the entire YAML layer is documentation overhead. The design argues "strictly better than unstructured Markdown" which is true, but "strictly better than _structured_ Markdown with defined sections and headers" is much less clear. The design replaced Forge's `## Status: DONE` (Markdown) with `completion: status: DONE` (YAML) — both are read by an LLM, both are "validated" by an LLM, neither has runtime enforcement. The format changed; the guarantee didn't.

  The schema definitions in `schemas.md` serve as useful documentation, but they could serve exactly the same purpose as "expected output format" sections in each agent's Markdown definition — which is what Anvil does, and Anvil is 419 lines for the whole thing.

- **Impact:** The design pays a complexity cost (10 schemas, schema evolution policy, producer/consumer tracking, YAML formatting issues, retry on malformed YAML) for a benefit that is, by its own admission, marginal over structured Markdown. The ROI of the YAML layer is negative when accounting for the format-specific failure modes it introduces.
- **Suggested fix:** This is not a "fix this" finding — the design's honesty is correct. The finding is: seriously consider whether the YAML layer justifies its cost. If the answer is "yes, the structural consistency benefits outweigh the format overhead," document this explicitly as a cost-benefit decision rather than a technical enforcement gain. If "no," simplify to two formats (SQL + Markdown) per Finding 5.

---

### [Finding 13]: FR-4.7 Revert Mechanism Is Still Unassigned After Two Revisions

- **Severity:** Medium
- **Area:** §Decision 6 (Verification-Replan Loop), FR-4.7
- **Issue:** FR-4.7 requires: "After 2 failed fix attempts for a verification check, the system MUST revert changes for that check and record the failure." The v1 ct-strategy review flagged this as architecturally unresolved. The v2 and v3 revisions mention it ("After 2 failed fix attempts for a specific verification check: revert changes for that check, record the failure in the ledger") but still don't assign which agent performs the revert:
  - The Implementer would be the natural choice, but it may have exited
  - The Verifier is restricted to read-only for source files ("MUST NOT modify source files")
  - The Orchestrator has `run_in_terminal` in v3, so it _could_ run `git checkout HEAD -- {files}`, but this violates the coordinator-only pattern

  This has been flagged across two review rounds and remains unresolved.

- **Impact:** Without a clear revert owner, the pipeline will leave broken code in place when verification fails twice, undermining the "never present broken code" guarantee that Anvil promises.
- **Suggested fix:** Assign reverts to the Implementer explicitly: when the Verifier reports `gate_status: failed` with specific failing checks after 2 attempts, the orchestrator dispatches the Implementer in "revert mode" with parameters indicating which files to revert. The Implementer has the tools (`run_in_terminal` for `git checkout`) and the file-write permissions needed. Alternatively, grant the Verifier limited write access specifically for `git checkout` operations on failing files.

---

### [Finding 14]: The Design Says "Evidence Gating" But Actually Means "Record Counting"

- **Severity:** Low
- **Area:** §Decision 6 (Evidence Gating Mechanism), Threat Model
- **Issue:** The design is honest about this in the "Honest limitation" callout, which is good. But the terminology throughout the rest of the document is misleading. "Evidence gating" connotes examining evidence and making a quality judgment. What actually happens is `SELECT COUNT(*) >= threshold` — counting that enough records exist, regardless of what those records say. A Verifier could INSERT 3 records that all say `passed=0` and the baseline gate (`COUNT(*) > 0`) would pass. The after-check gate does check `passed=1` but the point stands: the "gating" is quantity-based bookkeeping, not evidence assessment.

  The v3 improvement (orchestrator runs SQL queries independently) is genuine — it's better than trusting the Verifier's self-reported `gate_status`. But the _independent verification_ only verifies that records exist, not that verification was actually performed or that results are truthful. Two LLM agents agreeing on fabricated evidence is harder than one, but "harder" is not "prevented."

- **Impact:** Users (and future maintainers) may over-trust the "evidence gating" terminology and believe the pipeline has stronger quality assurance than it actually provides. The honest limitation callout helps but is buried in one section — the rest of the document uses "evidence gating" without qualification.
- **Suggested fix:** Rename to "evidence record verification" or "structural evidence gating" throughout, or add a parenthetical "(record-count-based)" wherever evidence gating is mentioned in pipeline flow descriptions. The honest limitation callout should be elevated to the Summary section or Decision 6 introduction, not buried after the gate table.

---

### [Finding 15]: What Forge Did Well That Was Lost

- **Severity:** Low
- **Area:** §Decision 2 (Agents Removed table), entire design
- **Issue:** A summary of capabilities lost from Forge that the design doesn't adequately address:
  1. **Post-Mortem agent** — Forge had a dedicated pipeline retrospective agent that analyzed what went wrong, measured performance, and suggested improvements. The design folds this into the Knowledge Agent, but knowledge capture and retrospective analysis are different tasks. Knowledge capture records facts; post-mortem asks "what went wrong and how do we prevent it?"
  2. **Memory merge as context compression** — The design correctly identifies memory merge as overhead (~12 dispatches). But merge served a purpose: it compressed multiple upstream outputs into a single curated summary. Late-pipeline agents like the Knowledge Agent must now read ALL upstream outputs (8+ YAML files + SQL). The `relevant_context` mechanism only helps Implementers, not late-stage agents.
  3. **Separate documentation writer** — For documentation-type tasks, a specialized agent likely produced better docs than an Implementer in "documentation mode." The merged approach assumes implementation and documentation are the same skill — they're not.
  4. **Three-state completion for all agents** — Several agents in the new design (Researcher, Spec, Designer, Implementer, Knowledge Agent) return only DONE | ERROR, losing the NEEDS_REVISION state that enabled targeted revision routing. Only the Planner, Verifier, and Adversarial Reviewer use NEEDS_REVISION.

- **Impact:** Each individual loss is minor. Collectively, they represent a regression in pipeline capabilities that the "9 agents, simpler pipeline" narrative glosses over.
- **Suggested fix:** (a) Restore NEEDS_REVISION for the Designer (needed for design revision loop — the pipeline already has one). (b) Document explicitly what capabilities were deliberately dropped and why. (c) Consider whether the Knowledge Agent can realistically perform both knowledge capture AND post-mortem analysis, or if post-mortem should be a separate non-blocking agent.

---

## Summary of Findings

| #   | Title                                    | Severity | Core Issue                                                                                     |
| --- | ---------------------------------------- | -------- | ---------------------------------------------------------------------------------------------- |
| 1   | Multi-model routing is unverified        | Critical | Entire adversarial review architecture rests on unverified platform assumption                 |
| 2   | Complexity relocated, not reduced        | High     | Verifier and Adversarial Reviewer are mega-agents hiding original multi-agent complexity       |
| 3   | Anvil's tight verify-fix loop lost       | High     | Multi-agent pipeline structurally cannot replicate Anvil's most powerful pattern               |
| 4   | Evidence Bundle dropped                  | High     | Key user-facing trust deliverable has no equivalent                                            |
| 5   | Tri-format storage over-engineered       | High     | SQLite + YAML + Markdown adds complexity Anvil solved with two formats                         |
| 6   | Pushback fires too late                  | High     | 4 researcher dispatches wasted before pushback evaluates request quality                       |
| 7   | Justification scoring is self-assessment | Medium   | No scoring rubric, no independent validation, no consequences for low confidence               |
| 8   | Git hygiene, recall, auto-commit dropped | Medium   | Three Anvil operational quality features silently omitted                                      |
| 9   | Orchestrator `run_in_terminal` reversal  | Medium   | Contradicts prior deliberate restriction without acknowledging tradeoff                        |
| 10  | Always-3 reviewers wastes dispatches     | Medium   | Diverges from spec; Anvil's task-sizing optimization dropped                                   |
| 11  | Perspective diversity lost               | Medium   | Model diversity ≠ perspective diversity; 7 specialized prompts → 1 generic prompt              |
| 12  | YAML layer cost vs. benefit              | Medium   | Design honestly admits no runtime guarantees; questions whether the YAML overhead is justified |
| 13  | FR-4.7 revert still unassigned           | Medium   | Flagged twice in prior reviews; still architecturally unresolved                               |
| 14  | "Evidence gating" overstates mechanism   | Low      | It's record counting, not evidence assessment                                                  |
| 15  | Forge capabilities lost                  | Low      | Post-mortem, context compression, doc specialization, NEEDS_REVISION for more agents           |

**Critical: 1 | High: 5 | Medium: 7 | Low: 2 | Total: 15**
