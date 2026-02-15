# Review: Comprehensive Forge Agent Improvements

**Overall Verdict: OK (Minor issues)**

The implementation of all 11 agent/prompt files is high-quality, comprehensive, and faithfully executes the design. Both P0 bugs are fixed, all three implementation clusters (TDD, Per-Task Agent Routing, Approval Gates) are atomically coordinated, and the cross-cutting patterns (Operating Rules, Anti-Drift Anchors, Completion Contracts) are consistently applied. The new agents are a substantial and genuine improvement over the originals.

---

## Changed Files Summary

| #   | File                                                                                        | Scope                                                                                                                                                                                                                              |
| --- | ------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | [orchestrator.agent.md](../../../NewAgentsAndPrompts/orchestrator.agent.md)                 | Full rewrite. 321 lines (was ~235). Added: three-state contracts, concurrency cap, per-task agent routing, approval gates, NEEDS_REVISION routing table, operating rules, anti-drift anchor.                                       |
| 2   | [researcher.agent.md](../../../NewAgentsAndPrompts/researcher.agent.md)                     | Full rewrite. ~140 lines (was ~95). Added: hybrid retrieval strategy, confidence metadata, decision log read, operating rules, anti-drift anchor.                                                                                  |
| 3   | [spec.agent.md](../../../NewAgentsAndPrompts/spec.agent.md)                                 | Rewrite. ~85 lines (was ~45). Added: structured edge cases, self-verification step, operating rules, anti-drift anchor.                                                                                                            |
| 4   | [designer.agent.md](../../../NewAgentsAndPrompts/designer.agent.md)                         | Rewrite. ~90 lines (was ~45). Added: security considerations, failure & recovery, self-verification, revision cycle input, operating rules, anti-drift anchor.                                                                     |
| 5   | [critical-thinker.agent.md](../../../NewAgentsAndPrompts/critical-thinker.agent.md)         | **Full rewrite.** ~95 lines (was ~30). Fixed P0 bugs: completion contract added, output file specified. Added: structured risk categories, structured workflow, operating rules, anti-drift anchor.                                |
| 6   | [planner.agent.md](../../../NewAgentsAndPrompts/planner.agent.md)                           | Rewrite. ~210 lines (was ~120). Added: mode detection, task size limits, pre-mortem analysis, plan validation, planning principles, agent field, operating rules, anti-drift anchor.                                               |
| 7   | [implementer.agent.md](../../../NewAgentsAndPrompts/implementer.agent.md)                   | **Full rewrite.** ~160 lines (was ~40). Replaced: prohibition-based workflow → TDD workflow. Added: TDD fallback, security rules, code quality principles, self-reflection, operating rules, anti-drift anchor.                    |
| 8   | [verifier.agent.md](../../../NewAgentsAndPrompts/verifier.agent.md)                         | Rewrite. ~155 lines (was ~110). Replaced: hardcoded .NET commands → technology-agnostic detection. Redefined role: sole test owner → integration-level verifier. Added: read-only enforcement, operating rules, anti-drift anchor. |
| 9   | [reviewer.agent.md](../../../NewAgentsAndPrompts/reviewer.agent.md)                         | Rewrite. ~180 lines (was ~65). Added: security review (secrets + OWASP), tiered review depth, read-only enforcement, decision log write, quality standard, self-reflection, operating rules, anti-drift anchor.                    |
| 10  | [feature-workflow.prompt.md](../../../NewAgentsAndPrompts/feature-workflow.prompt.md)       | Updated. Added: concurrency cap rule, per-task agent routing rule, APPROVAL_MODE variable with variables table.                                                                                                                    |
| 11  | [documentation-writer.agent.md](../../../NewAgentsAndPrompts/documentation-writer.agent.md) | **New file.** ~80 lines. Full agent: role, capabilities, read-only enforcement, workflow, completion contract, anti-drift anchor.                                                                                                  |

---

## Acceptance Criteria Verification

| AC    | Description                      | Status   | Notes                                                                                                                                                                                          |
| ----- | -------------------------------- | -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| AC-1  | All 11 files exist and non-empty | **PASS** | All 11 files present in `NewAgentsAndPrompts/`                                                                                                                                                 |
| AC-2  | Critical-thinker bugs fixed      | **PASS** | `## Completion Contract` present with DONE/NEEDS_REVISION/ERROR. Output `design_critical_review.md` explicitly specified.                                                                      |
| AC-3  | TDD cluster consistency          | **PASS** | Orchestrator rule 6 states TDD. Implementer has TDD workflow, no "Do NOT build/run" remnants. Verifier role is "integration-level."                                                            |
| AC-4  | Concurrency cap                  | **PASS** | Orchestrator rule 7: max 4, sub-wave splitting. Prompt reinforces max 4.                                                                                                                       |
| AC-5  | Planner improvements             | **PASS** | Pre-mortem analysis, task size limits (3 files/2 deps/500 lines/medium), mode detection (initial/replan/extension).                                                                            |
| AC-6  | Security thread                  | **PASS** | Implementer: security rules. Reviewer: secrets scan + OWASP. Designer: security considerations section.                                                                                        |
| AC-7  | Technology portability           | **PASS** | No hardcoded project names. Verifier has build system detection table (8 ecosystems + non-exhaustive fallback). "dotnet" only appears as one entry in the generic detection table.             |
| AC-8  | Anti-drift anchors               | **PASS** | All 10 agents have `## Anti-Drift Anchor` with correct `**REMEMBER:**` text and `Stay as <role>`.                                                                                              |
| AC-9  | Cross-cutting consistency        | **PASS** | All agents have DONE/ERROR contracts. Three agents add NEEDS_REVISION (critical-thinker, verifier, reviewer). Operating Rules block present in all 10. No contradictions found between agents. |
| AC-10 | Per-task agent routing           | **PASS** | Planner defines `agent` field in task files. Orchestrator rule 8 + Step 5.2 routing logic. Prompt documents routing. Valid agents: `implementer`, `documentation-writer`.                      |
| AC-11 | Approval gates                   | **PASS** | Orchestrator rule 9 (experimental) + Steps 1.2a, 4a. Prompt defines `{{APPROVAL_MODE}}` with default `false`. Fallback for non-interactive environments.                                       |
| AC-12 | Documentation writer             | **PASS** | File exists with full structure. Not referenced in core pipeline — only via per-task routing.                                                                                                  |

---

## Issues & Severity

### Minor Issues

#### M-1: Orchestrator Step 1.1 lists `NEEDS_REVISION` for researchers

**File:** [orchestrator.agent.md](../../../NewAgentsAndPrompts/orchestrator.agent.md#L108)

Step 1.1 says: _"Wait for ALL three to return `DONE:`, `NEEDS_REVISION:`, or `ERROR:`."_ However, the researcher's completion contract only defines `DONE:` and `ERROR:`. The NEEDS_REVISION Routing Table and the Orchestrator Expectations Table both correctly mark researcher NEEDS_REVISION as "N/A."

The step description text is unnecessary broad. An LLM following these instructions might look for NEEDS_REVISION handling logic for researchers and find none, causing confusion.

**Suggested Fix:** Change the Step 1.1 wait description to: _"Wait for ALL three to return `DONE:` or `ERROR:`."_ Apply the same fix to Step 1.2 (synthesis researcher). Leave Steps 3b, 6, and 7 with the three-state wording since those agents do use NEEDS_REVISION.

**Severity:** Minor — cosmetic text inconsistency; functionally harmless since the case never arises.

---

#### M-2: Verifier `Outputs` section contradicts Operating Rule 4 (file boundaries)

**File:** [verifier.agent.md](../../../NewAgentsAndPrompts/verifier.agent.md#L28)

The `## Outputs` section lists only `docs/feature/<feature-slug>/verifier.md`. But the `## Read-Only Enforcement` section also permits: _"Task file status updates (marking tasks as verified/partially-verified/failed)."_ This means the verifier writes to `tasks/*.md` files, which are NOT in its Outputs section.

Operating Rule 4 states: _"Only write to files listed in the Outputs section."_ This creates a self-contradiction within the verifier agent definition.

**Suggested Fix:** Add `docs/feature/<feature-slug>/tasks/*.md (status updates only)` to the Outputs section.

**Severity:** Minor — the Read-Only Enforcement section makes the intent clear, but the contradiction could confuse an LLM interpreting rules strictly.

---

#### M-3: Documentation writer tool preference gap for `get_changed_files`

**File:** [documentation-writer.agent.md](../../../NewAgentsAndPrompts/documentation-writer.agent.md#L50)

Workflow step 5 references `get_changed_files` (or equivalent) for delta-only verification, but the Operating Rules tool preferences (rule 5) don't mention this tool. All other agents list every tool they use in rule 5.

**Suggested Fix:** Update rule 5 to: _"Use `semantic_search` and `grep_search` for code discovery. Use `read_file` for targeted examination. Use `get_changed_files` for delta verification (fall back to `grep_search` on recent commits if unavailable). Never use tools that modify source code."_

**Severity:** Minor — the workflow step is clear enough, but tool preference consistency matters for the canonical standard.

---

#### M-4: Critical-thinker workflow doesn't mention `decisions.md`

**File:** [critical-thinker.agent.md](../../../NewAgentsAndPrompts/critical-thinker.agent.md)

The design's Pattern 5 (`decisions.md` Lifecycle) states: _"Critical-thinker may also reference it to check for conflicts."_ However, the critical-thinker's workflow contains no instruction to read `decisions.md`, nor is it listed in the Inputs section.

The researcher does correctly reference it: _"If `decisions.md` exists in the documentation structure, read it."_

**Suggested Fix:** Add to the critical-thinker's workflow step 4 (or as a new sub-step): _"If `decisions.md` exists, check for conflicts between prior architectural decisions and the current design."_ Also add `decisions.md (if exists — optional)` to the Inputs section.

**Severity:** Minor — the critical-thinker can still identify conflicts by examining the design itself, but explicitly reading prior decisions would improve coverage.

---

#### M-5: Implementer self-reflection references "High" effort tasks that cannot exist

**File:** [implementer.agent.md](../../../NewAgentsAndPrompts/implementer.agent.md#L137)

The self-reflection step header says _"Medium/High effort tasks only"_ but the planner's task size limits cap effort at "Medium" — High effort tasks must be decomposed. The "High" condition is dead code.

**Suggested Fix:** Change to _"Medium effort tasks only"_ or keep as-is for future-proofing. Not functionally harmful.

**Severity:** Minor — no functional impact; the condition still correctly triggers for Medium tasks.

---

#### M-6: Orchestrator step descriptions omit `initial-request.md` from per-step input lists

**File:** [orchestrator.agent.md](../../../NewAgentsAndPrompts/orchestrator.agent.md)

Step 0 establishes the blanket rule: _"This file MUST be provided as an input to every subagent."_ However, individual step descriptions (Steps 2, 3, 3b, 4) list only step-specific inputs without `initial-request.md`. For example, Step 3b says _"Input: design.md, feature.md"_ but the critical-thinker also requires `initial-request.md`.

This is covered by Step 0's blanket rule, so the behavior is technically correct. However, an LLM reading Step 3b in isolation might not recall Step 0's rule and fail to pass the file.

**Suggested Fix:** No change strictly required — the blanket rule is clear. Optionally, add a parenthetical like _(+ initial-request.md per Step 0)_ to each step for safety. Given the orchestrator's length (321 lines), the LLM may lose early context.

**Severity:** Minor — Step 0 covers this, but the distance between Step 0 and later steps risks LLM context loss.

---

#### M-7: Feature spec constraint #4 is out of sync with design and implementation

**File:** [feature.md](feature.md) — Constraint #4

The feature spec states: _"Completion contract: All agents must use `DONE:`/`ERROR:` text-based completion (not JSON, not three-state)."_ The design explicitly overrode this constraint (documented in the Review Response section, issue B-1) to adopt three-state completion (`NEEDS_REVISION:`). The implementation correctly follows the design.

**Suggested Fix:** Update `feature.md` constraint #4 to reflect the design decision: _"Completion contract: All agents must use `DONE:`/`NEEDS_REVISION:`/`ERROR:` text-based completion (not JSON). Only critical-thinker, reviewer, and verifier use `NEEDS_REVISION:`."_

**Severity:** Minor — the design is the governing document and is correctly followed. The spec is a historical artifact that ideally should have been updated.

---

### Observations (Non-Issues)

#### O-1: Operating Rules "Global Rule 4" cross-reference

All agents' Operating Rules reference _"The orchestrator also retries entire agent invocations once (Global Rule 4)"_ — but "Global Rule 4" is an orchestrator-internal concept. For non-orchestrator agents, this is informational context. Since the text clearly explains the behavior without requiring the agent to act on it, this is acceptable.

#### O-2: Orchestrator NEEDS_REVISION Routing Table + Expectations Table redundancy

The orchestrator contains both a NEEDS_REVISION Routing Table and an Orchestrator Expectations Per Agent table that overlap significantly. This is intentional redundancy — it reinforces routing logic through multiple representations, which is beneficial for LLM comprehension. The tables are consistent with each other.

#### O-3: `detailed thinking on` directive is experimental and labeled

All 10 agents include `Use detailed thinking to reason through complex decisions before acting.` with `<!-- experimental: model-dependent -->` HTML comments. The design explicitly marks this as experimental with removal instructions. The HTML comment makes batch removal trivial via search. Well-handled.

#### O-4: Three-state completion is well-contained

Only 3 of 11 agents use `NEEDS_REVISION:` (critical-thinker, reviewer, verifier). The remaining 8 use only `DONE:`/`ERROR:`. This keeps complexity bounded while enabling the lightweight revision paths that eliminate expensive replan cycles for minor fixes. The orchestrator's routing tables clearly document all three-state interactions.

---

## Diff Highlights — Key Improvements Over Originals

### Most Impactful Changes

1. **Critical-thinker (P0 bug fixes + complete rewrite):** The original was a 30-line conversational prompt with no completion contract, no output file specification, and vague "ask why" instructions. The new version is a 95-line structured agent with risk categories, likelihood/impact assessment, completion contract, and specific output format. This is the single largest quality improvement — the original was effectively broken in the pipeline.

2. **Implementer (TDD + security rules):** Transformed from a 40-line passive agent that was explicitly prohibited from running tests into a 160-line TDD-driven agent with security awareness, code quality principles, and self-reflection. The "Do NOT build" / "Do NOT run tests" prohibitions are completely eliminated.

3. **Verifier (technology portability):** Hardcoded `dotnet build TourneyPal.sln` and `dotnet test tests/TourneyPal.Tests` replaced with an 8-ecosystem detection table plus non-exhaustive fallback. The agent now works with any project, not just one specific .NET solution.

4. **Reviewer (security thread):** Expanded from a generic 65-line review agent to a 180-line security-aware reviewer with OWASP Top 10 checklist, tiered review depth, secrets scanning, and decision log maintenance.

5. **Orchestrator (concurrency + routing):** Added sub-wave splitting for >4 concurrent agents, per-task agent routing, three-state completion handling with routing table, approval gates, and comprehensive agent expectation documentation.

### Cross-Cutting Improvements

- **Operating Rules:** Consistent error handling across all 10 agents with retry budgets and deterministic failure exclusion.
- **Anti-Drift Anchors:** Final-position reinforcement blocks in all agents to combat LLM context drift.
- **Self-verification/reflection:** Spec, designer, critical-thinker, implementer, and reviewer all self-check before returning.
- **Security thread:** Three-layer defense: designer (design-level), implementer (code-level), reviewer (audit-level).

---

## Testing Impact

### Tests to Run (from feature.md TS-1 through TS-15)

| Test  | Description             | Expected Result                                                                                                |
| ----- | ----------------------- | -------------------------------------------------------------------------------------------------------------- |
| TS-1  | File existence          | All 11 files exist and non-empty — **PASS**                                                                    |
| TS-2  | Critical-thinker bugs   | Completion contract exists, `design_critical_review.md` specified — **PASS**                                   |
| TS-3  | TDD cluster consistency | Implementer has TDD, no "Do NOT" remnants, verifier is integration-level, orchestrator states TDD — **PASS**   |
| TS-4  | Concurrency cap         | Orchestrator: "4", sub-wave splitting; Prompt: "max 4" — **PASS**                                              |
| TS-5  | Planner completeness    | Pre-mortem, size limits, mode detection all present — **PASS**                                                 |
| TS-6  | Security thread         | All three agents cover security in their domain — **PASS**                                                     |
| TS-7  | Technology portability  | No hardcoded project names; detection table present — **PASS**                                                 |
| TS-8  | Anti-drift anchors      | All 10 agents have anchors with "Stay as" — **PASS**                                                           |
| TS-9  | Completion contracts    | All 10 agents have DONE/ERROR — **PASS**                                                                       |
| TS-10 | Per-task agent routing  | Planner `agent` field, orchestrator routing, prompt documentation — **PASS**                                   |
| TS-11 | Approval gates          | Orchestrator APPROVAL_MODE logic, prompt variable, default autonomous — **PASS**                               |
| TS-12 | Documentation writer    | Exists, not in core pipeline, has full structure — **PASS**                                                    |
| TS-13 | Hybrid retrieval        | Researcher prescribes semantic_search → grep_search → merge → read_file → file_search — **PASS**               |
| TS-14 | Cross-agent consistency | No contradictions found (except minor items noted above) — **PASS**                                            |
| TS-15 | Error handling          | Operating Rules present in all agents with transient/persistent/security/missing-context categories — **PASS** |

### Test Additions Suggested

None — these are agent definition files, not code. The TS-1 through TS-15 test scenarios from the feature spec serve as the verification checklist and all pass.

---

## Architectural Decisions

No new architectural decisions logged — the implementation faithfully follows the design. The design's Review Response section already documents all design-level decisions made in response to the critical review, including the adoption of three-state completion.

---

## Conformance to .github Instructions

No `.github/instructions/` directory currently exists in the workspace. The reviewer agent is configured to update `.github/instructions/*.instructions.md` when it identifies new conventions — this is forward-compatible. No instruction updates are needed at this time since the agents are output to `NewAgentsAndPrompts/` (not `.github/agents/`).

When these agents are eventually deployed to `.github/agents/`, a `.github/instructions/` directory may be created to codify project conventions discovered during reviews.

---

## Owners & Next Steps

| Priority | Action                                                                      | Owner            | Files                                                                                                                                                              |
| -------- | --------------------------------------------------------------------------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Optional | Fix M-1: Remove `NEEDS_REVISION` from orchestrator Steps 1.1/1.2 wait text  | Implementer      | [orchestrator.agent.md](../../../NewAgentsAndPrompts/orchestrator.agent.md#L108)                                                                                   |
| Optional | Fix M-2: Add task files to verifier Outputs section                         | Implementer      | [verifier.agent.md](../../../NewAgentsAndPrompts/verifier.agent.md#L28)                                                                                            |
| Optional | Fix M-3: Add `get_changed_files` to doc-writer tool preferences             | Implementer      | [documentation-writer.agent.md](../../../NewAgentsAndPrompts/documentation-writer.agent.md#L30)                                                                    |
| Optional | Fix M-4: Add `decisions.md` reference to critical-thinker                   | Implementer      | [critical-thinker.agent.md](../../../NewAgentsAndPrompts/critical-thinker.agent.md)                                                                                |
| Optional | Fix M-7: Sync feature.md constraint #4 with design decision                 | Spec Agent       | [feature.md](feature.md)                                                                                                                                           |
| Future   | Deploy to `.github/agents/` when ready                                      | Human maintainer | All 11 files                                                                                                                                                       |
| Future   | Validate `{{APPROVAL_MODE}}` against VS Code Copilot agent protocol         | Human maintainer | [orchestrator.agent.md](../../../NewAgentsAndPrompts/orchestrator.agent.md), [feature-workflow.prompt.md](../../../NewAgentsAndPrompts/feature-workflow.prompt.md) |
| Future   | Test `detailed thinking on` directive with target models; remove if harmful | Human maintainer | All 10 agent files                                                                                                                                                 |

---

## Checklist / Acceptance

- [x] All 11 files exist and are non-empty (AC-1)
- [x] P0 bugs fixed: critical-thinker completion contract + output spec (AC-2)
- [x] TDD cluster consistent across orchestrator/implementer/verifier (AC-3)
- [x] Concurrency cap in orchestrator + prompt (AC-4)
- [x] Planner has pre-mortem, size limits, mode detection (AC-5)
- [x] Security thread across designer/implementer/reviewer (AC-6)
- [x] Verifier is technology-agnostic (AC-7)
- [x] All 10 agents have anti-drift anchors (AC-8)
- [x] All agents have consistent completion contracts (AC-9)
- [x] Per-task agent routing coordinated across planner/orchestrator/prompt (AC-10)
- [x] Approval gates conditional on APPROVAL_MODE (AC-11)
- [x] Documentation writer exists and not in core pipeline (AC-12)
- [x] All TS-1 through TS-15 test scenarios pass
- [x] No blocking issues found
- [ ] Optional: Address M-1 through M-7 minor issues (non-blocking)
