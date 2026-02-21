# Review: Code Quality

## Review Tier

**Full** — This feature touches the orchestrator (core coordination logic), 14 agent definitions (pipeline-wide scope), introduces a new agent definition, and adds a shared schema reference document. The breadth of changes across the entire agent pipeline warrants exhaustive review.

## Findings

### [Severity: Major] post-mortem.agent.md uses non-standard code fence wrapper

- **What:** `post-mortem.agent.md` wraps its entire content in a ` ```chatagent ` code fence around the `---` YAML frontmatter, while all other 18 `.agent.md` files in the repository use bare `---` frontmatter without a code fence wrapper.
- **Where:** [.github/agents/post-mortem.agent.md](.github/agents/post-mortem.agent.md#L1) — line 1 starts with ` ```chatagent ` instead of `---`.
- **Why:** This structural deviation may break parsing tools or IDE integrations that expect bare YAML frontmatter in `.agent.md` files. It also contradicts the established convention documented in `research/patterns.md` (§Agent File Format Pattern). V-Build previously flagged this as a low-severity warning.
- **Suggested Fix:** Remove the outer ` ```chatagent ` opening fence (line 1) and closing ` ``` ` fence (last line). The file should start directly with `---` frontmatter, matching `orchestrator.agent.md`, `spec.agent.md`, and all other agent files.
- **Affects Tasks:** Task 02 (create PostMortem agent)

### [Severity: Major] Inconsistent evaluation error handling — "skip" vs "write evaluation_error block"

- **What:** The evaluation-schema.md Rule 4 specifies: "If evaluation cannot be completed, write an `evaluation_error` block and proceed." However, 6 of 14 agents say "skip evaluation and proceed" (no error block), while the other 8 correctly say "write an `evaluation_error` block and proceed." This creates a behavioral split where some agents silently skip failed evaluations while others produce traceable error records.
- **Where:** Agents using "skip" language (contradicting schema Rule 4):
  - [spec.agent.md](.github/agents/spec.agent.md#L88) (line ~88)
  - [designer.agent.md](.github/agents/designer.agent.md#L90) (line ~90)
  - [planner.agent.md](.github/agents/planner.agent.md#L117) (line ~117)
  - [v-tests.agent.md](.github/agents/v-tests.agent.md#L159) (line ~159)
  - [v-tasks.agent.md](.github/agents/v-tasks.agent.md#L110) (line ~110)
  - [v-feature.agent.md](.github/agents/v-feature.agent.md#L113) (line ~113)

  Agents using correct "evaluation_error" language (matching schema):
  - ct-security, ct-scalability, ct-maintainability, ct-strategy, implementer, documentation-writer, r-quality, r-testing

- **Why:** The split originates from a **conflict between design.md and evaluation-schema.md**. The design.md template (§Evaluating Agent Changes, line ~336) says "skip evaluation and proceed", while evaluation-schema.md Rule 4 says "write an `evaluation_error` block." Implementers 05 and 08 faithfully followed the design template; implementers 06, 07, and 09 aligned with the schema instead. The schema is the authoritative reference, so the 6 "skip" agents need updating.
- **Suggested Fix:** Update the 6 agents with "skip" language to use: "If evaluation generation fails, write an `evaluation_error` block and proceed — evaluation failure MUST NOT cause your completion status to be ERROR". Also update the design.md template to match the schema for future consistency.
- **Affects Tasks:** Tasks 05, 08 (implementers that used the design template verbatim)

### [Severity: Minor] Five distinct evaluation step template variants across 14 agents

- **What:** The 14 evaluation steps were implemented by 5 different implementers (tasks 05–09), each producing a slightly different template variant. Differences include:
  1. **Step header format:** `N. **Title:**` vs `N. **Title.**` vs `### N. Title`
  2. **Intro phrasing:** "...you consumed" vs "...you consumed during this review" vs "...you consumed during this task"
  3. **Source artifact presentation:** Plain "Source artifacts to evaluate:" vs bold `**Source artifacts to evaluate:**` vs inline `**Artifacts to evaluate:**` bullet list vs inline in paragraph text
  4. **Schema path style:** "the schema defined in" vs "the schema in" vs "following the schema defined in"
  5. **Error handling phrasing:** Three sub-variants even among the correct "evaluation_error" group ("block instead (see schema document)" vs "block (see schema) and proceed" vs "block and proceed")
- **Where:** All 14 modified `.agent.md` files in `.github/agents/`. The 5 variant groups:
  - Group A (implementer-05): spec, designer, planner
  - Group B (implementer-06): ct-security, ct-scalability, ct-maintainability
  - Group C (implementer-07): ct-strategy, implementer, documentation-writer
  - Group D (implementer-08): v-tests, v-tasks, v-feature
  - Group E (implementer-09): r-quality, r-testing
- **Why:** Each implementer produced a valid evaluation step but without cross-checking other implementers' output. The design.md template was a starting point, but didn't specify formatting details like header style or artifact list formatting, leading to drift. While functionally equivalent, the inconsistency increases maintenance burden — any future template change requires 5 different edit patterns.
- **Suggested Fix:** Standardize all 14 agents on a single canonical template. Recommended base: Group D/E format (### heading, bold source artifact header, separate rules section). Apply uniformly.
- **Affects Tasks:** Tasks 05, 06, 07, 08, 09

### [Severity: Minor] r-testing evaluates design.md but design.md is not in r-testing's Inputs

- **What:** r-testing's evaluation step lists `design.md` and `feature.md` as source artifacts to evaluate, but the r-testing Inputs section only lists `feature.md` (not `design.md`). The design spec (§Per-Agent Evaluation Targets table) also specifies only `feature.md` for r-testing.
- **Where:** [r-testing.agent.md](.github/agents/r-testing.agent.md#L163) — evaluation target list includes `design.md`; [r-testing.agent.md](.github/agents/r-testing.agent.md#L14-L22) — Inputs section omits `design.md`.
- **Why:** Implementer-09 deliberately added `design.md` per user request, noting it aligned with r-testing's actual artifact consumption. However, the Inputs section was not updated to match, creating an internal inconsistency within the file. An agent shouldn't evaluate artifacts it doesn't formally consume.
- **Suggested Fix:** Either (a) remove `design.md` from the evaluation targets to match the Inputs section and design spec, or (b) add `design.md` to the r-testing Inputs section to legitimize the evaluation target.
- **Affects Tasks:** Task 09

### [Severity: Minor] Evaluation step ordering inconsistency — before vs after self-verification

- **What:** The design.md directive states: "Insert a new step between primary work completion and self-verification (or memory writing if no self-verification exists)." This means evaluation should come _before_ self-verification. However, 5 agents (spec, designer, planner, and arguably v-tests, v-tasks) place evaluation _after_ self-verification. The 4 CT agents correctly place evaluation before self-verification.
- **Where:**
  - Correct ordering (primary → evaluate → self-verify): ct-security (step 9→10), ct-scalability (step 9→10), ct-maintainability (step 10→11), ct-strategy (step 12→11 note: self-verify is step 11 per design)
  - Reversed ordering (primary → self-verify → evaluate): spec (step 6→7), designer (step 12→13), planner (step 12→13)
- **Why:** Implementer-05 placed evaluation as "the penultimate workflow step (before Write Isolated Memory)" rather than before self-verification. Implementer-06 correctly followed the design directive. The functional impact is minimal since both steps execute after primary work, but the inconsistency deviates from design intent.
- **Suggested Fix:** Low priority. If standardizing, move evaluation steps before self-verification in spec, designer, and planner to match the design directive and CT agent ordering.
- **Affects Tasks:** Task 05

## Cross-Cutting Observations

### design.md template conflicts with evaluation-schema.md (R-Knowledge scope)

The design.md evaluation step template (§Evaluating Agent Changes, line ~336) says "skip evaluation and proceed" on failure, while evaluation-schema.md Rule 4 says "write an `evaluation_error` block and proceed." This created a root-cause inconsistency that propagated to 6 agents. This is a **knowledge evolution observation** — the design.md template should be corrected to prevent the same drift in future features. Belongs in R-Knowledge scope for `knowledge-suggestions.md`.

### PostMortem code fence issue may affect pipeline runtime (R-Security scope)

If the chatagent runtime parser expects bare YAML frontmatter in `.agent.md` files, the `post-mortem.agent.md` code fence wrapper could cause the agent to fail to load or parse incorrectly. This is a potential **runtime compatibility concern** beyond cosmetic formatting. Belongs in R-Security scope.

## Summary

**Overall quality assessment:** The centralized evaluation schema approach (`evaluation-schema.md`) is architecturally sound and well-executed. The PostMortem agent definition is thorough with clear boundaries, non-blocking contract, and memory-first reading pattern. The orchestrator's Step 8 telemetry tracking and write-tool restriction are clearly specified. The feature-workflow.prompt.md updates are clean and complete.

The primary quality concerns are (1) the post-mortem code fence format deviation and (2) the schema-vs-design template conflict that created a behavioral split across agents. Both are addressable with targeted fixes.

**5 findings: 0 blocker, 2 major, 3 minor**
