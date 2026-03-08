# Adversarial Review: Design — Security Sentinel

## Review Metadata

- **Perspective:** security-sentinel
- **Risk Level:** 🟡
- **Round:** 1
- **Run ID:** 2026-03-08T12:00:00Z

---

## Security Analysis

**Category Verdict:** needs_revision

### Finding S-1: Implementer `run_in_terminal` scope is instruction-only

- **Severity:** Major
- **Description:** The Implementer agent has `run_in_terminal` in its platform-enforced `tools` list but is only constrained to "unit test commands" by instruction text. The VS Code platform cannot distinguish `dotnet test` from arbitrary commands such as `curl`, `rm`, or `Invoke-WebRequest`. This means the Implementer could execute any terminal command on the developer's machine despite the design's intent to restrict it to unit tests. The design acknowledges this ("The platform can't distinguish `dotnet test` from `dotnet run`") but the gap persists.
- **Affected artifacts:** design-output.yaml D-10 (component_design → implementer → frontmatter), design.md "Per-Agent Tool Access" table, design.md "Security Considerations" → "Remaining Instruction-Level Restrictions"
- **Recommendation:** Add an explicit constraint in global-rules.md or the Implementer's Constraints section listing a **command allowlist pattern** (e.g., `^dotnet test`, `^npm test`, `^pytest`, `^go test`) that the orchestrator or Tester can audit in implementation reports. This doesn't provide platform enforcement but creates an auditable trail. Reference: the existing system's `tier5_command_allowlist` pattern in tool-access-matrix.md.
- **Evidence:** design-output.yaml component_design.implementer.frontmatter.tools includes `run_in_terminal`. design.md "Security Considerations" states: "Implementer `run_in_terminal` scope: Has terminal access but constrained to unit test commands by instructions. The platform can't distinguish `dotnet test` from `dotnet run`. This remains an instruction-enforced restriction."

### Finding S-2: `fetch_webpage` toggle is instruction-only in autonomous mode

- **Severity:** Major
- **Description:** D-10 selects instruction-level toggle for web research. `fetch_webpage` is permanently in the `tools` list of Researcher, Architect, and Implementer. In interactive mode, VS Code prompts per invocation (secondary control). In autonomous mode, the only guard is the `web_research_enabled=false` instruction parameter. If an agent ignores this instruction, it can fetch arbitrary URLs without user approval. While `fetch_webpage` is read-only (no POST), it could be used for SSRF-like behavior (probing internal network endpoints) or leaking context via URL parameters.
- **Affected artifacts:** design-output.yaml D-10 (Web Research Integration), design.md "Security Considerations" → "Web Research Security", component_design for researcher/architect/implementer frontmatter
- **Recommendation:** The design should explicitly document the autonomous-mode risk as an accepted residual risk with severity rating, OR specify that autonomous mode disables `fetch_webpage` by using separate agent variants (D-10 alt-2 as fallback). At minimum, add a constraint that agents MUST log all `fetch_webpage` invocations in their output YAML, creating an audit trail verifiable by Code Review.
- **Evidence:** design-output.yaml D-10 selected alt-1: "fetch_webpage in tools list + instruction-level toggle." D-10 confidence: "Medium." design.md states: "In autonomous mode, web research is disabled by the `web_research_enabled` parameter" — but this is instruction-enforced only.

### Finding S-3: Knowledge agent executes post-review with write capabilities

- **Severity:** Major
- **Description:** The Knowledge agent runs in Step 8 (Completion), which is AFTER Code Review (Step 7). It has `create_file` and `memory` tools. While its instructions say "MUST NOT modify .agent.md files," this is instruction-enforced only. If the Knowledge agent creates unexpected files (e.g., in the `v2/.github/agents/` directory or `/memories/repo/`), no automated step verifies its output before the Orchestrator executes `git add . + git commit`. The `create_file` tool's documentation says "Never use this tool to edit a file that already exists" but this is also instruction-enforced, not platform-enforced. Post-review write access creates an unreviewed mutation path.
- **Affected artifacts:** design-output.yaml pipeline_design Step 8, component_design.knowledge.frontmatter.tools (includes `create_file`, `memory`), design.md "Pipeline Design" → "Step 8: Completion"
- **Recommendation:** Add a post-Knowledge verification micro-step where the Orchestrator checks the git diff (`git diff --staged`) before committing, ensuring only expected output paths (evidence-bundle.yaml, knowledge-output.yaml, instruction-recommendations.md) are staged. Alternatively, specify an explicit allowlist of output paths the Knowledge agent may create, and have the Orchestrator validate `git diff --name-only` against this allowlist before `git commit`.
- **Evidence:** design-output.yaml pipeline_design step 8: "Knowledge agent: produces evidence-bundle.yaml... Orchestrator: git add + commit." No verification step between Knowledge output and git commit. FR-10.3 states: "The Knowledge agent MUST NOT modify copilot-instructions.md or agent definition files directly" — instruction constraint only.

### Finding S-4: Orchestrator has `replace_string_in_file` without justification

- **Severity:** Minor
- **Description:** The Orchestrator's tool list includes `replace_string_in_file`, granting it the ability to modify any existing file content. The Orchestrator's stated responsibilities (pipeline coordination, DAG dispatch, routing, git ops) don't clearly require editing existing files — `create_file` suffices for output generation, and git operations use `run_in_terminal`. The "No Runtime Agent Modification" global rule prohibits agents from modifying `.agent.md` files, but `replace_string_in_file` makes this possible if the Orchestrator drifts.
- **Affected artifacts:** design-output.yaml component_design.orchestrator.frontmatter.tools, design.md "Per-Agent Tool Access" table
- **Recommendation:** Document why the Orchestrator needs `replace_string_in_file` (e.g., updating copilot-instructions.md during promotion). If no justification exists, remove it from the Orchestrator's tools list.
- **Evidence:** design-output.yaml component_design.orchestrator.responsibilities lists 7 items — none obviously require editing existing files. design.md tool matrix shows only Orchestrator and Implementer have `replace_string_in_file`.

---

## Architecture Analysis

**Category Verdict:** needs_revision

### Finding A-1: No explicit trust boundary analysis for code-execution system

- **Severity:** Major
- **Description:** The design describes 8 agents that autonomously execute code, read/write files, run terminal commands, and fetch web content. Yet there is no explicit trust boundary diagram or analysis. For a system where agents can execute arbitrary commands (`run_in_terminal`), the design should identify: (1) which agents are trusted vs. least-privileged, (2) data flow paths between agents (all through file system via the Orchestrator), (3) potential privilege escalation paths (e.g., Implementer writes code → Tester executes it, transferring trust), (4) poisoned-data flows (e.g., research output influencing architect decisions). The "Security Considerations" section is a good start but covers only tool access and web research, not the systemic trust model.
- **Affected artifacts:** design.md "Security Considerations" section (incomplete coverage), design-output.yaml (no trust boundary section)
- **Recommendation:** Add a "Trust Model" subsection to Security Considerations that explicitly maps: (a) agent trust tiers (Orchestrator=coordinator, Implementer/Tester=execution, others=read-mostly), (b) data flow trust transitions (implementer code → tester execution = trust elevation), (c) accepted residual risks per tier. This doesn't add implementation complexity — it's documentation that guides future modifications.
- **Evidence:** design.md "Security Considerations" contains 3 subsections: "Tool Access Security," "Remaining Instruction-Level Restrictions," "Web Research Security." No trust boundary analysis, no data flow security analysis, no privilege escalation assessment. For comparison, the tool-access-matrix.md in the current system has 5 tiers with explicit security rationale per tier.

### Finding A-2: No integrity verification on routing artifacts

- **Severity:** Minor
- **Description:** The Orchestrator routes between steps solely by reading `completion.status` from YAML files and checking file existence (D-11). There is no integrity check on file content — an agent could write `status: DONE` without actually completing its work. While LLM agents aren't adversarial by design, instruction drift or hallucination could produce false completion signals. The current system has the same trust model (agents self-report), so this is not a regression — but the design opportunity to improve is missed.
- **Affected artifacts:** design-output.yaml D-11, state_management.routing_mechanism, design.md "State Management" section
- **Recommendation:** No immediate action required. Consider adding a lightweight content validation for critical gates: e.g., for the Research gate, check that research/\*.yaml files are non-empty with YAML parse success — not just existence. This adds ~5 lines of orchestrator logic.
- **Evidence:** design-output.yaml D-11 alt-1: "Completion contract + file existence check." Alt-3 (content validation) was explicitly rejected: "Orchestrator becomes a validator (scope creep)." EC-1 handles malformed YAML but not semantically empty or false-positive completions.

---

## Correctness Analysis

**Category Verdict:** needs_revision

### Finding C-1: Undefined orchestrator behavior on malformed YAML status values

- **Severity:** Major
- **Description:** The completion contract specifies `status: "DONE" | "NEEDS_REVISION" | "ERROR"` but the design doesn't specify what happens when the Orchestrator reads an unexpected value (e.g., `status: "DONE_PARTIAL"`, `status: true`, missing `status` field). EC-1 handles "malformed YAML" (parse failure) but not syntactically valid YAML with unexpected semantic values. If an unexpected status is treated as anything other than ERROR, it could cause the Orchestrator to skip mandatory steps (e.g., flowing past Code Review).
- **Affected artifacts:** design-output.yaml schemas.completion_contract, pipeline_design (routing logic per step), design.md "Completion Contract" definition
- **Recommendation:** Add to global-rules.md: "Any completion status value other than exactly DONE, NEEDS_REVISION, or ERROR MUST be treated as ERROR." This one-line rule prevents undefined routing behavior.
- **Evidence:** design-output.yaml schemas.completion_contract defines `status: "DONE" | "NEEDS_REVISION" | "ERROR"` but no validation rule. EC-1 covers "malformed YAML" only. No other edge case addresses status value validation. design.md "Failure & Recovery" table doesn't include "unexpected status value" as a failure mode.

### Finding C-2: Quick-fix prompt step-skipping behavior undefined — risk of bypassing mandatory Code Review

- **Severity:** Major
- **Description:** The design lists `quick-fix.prompt.md` (~25 lines) in the file inventory as a "Simplified pipeline for trivial fixes." The design does NOT specify which pipeline steps the quick-fix prompt includes or skips. Given repository history (the fast-track pipeline bypass of Step 7 was caught as a Critical finding in the agent-pipeline-improvements feature, leading to D-14 establishing the immutable minimum step set), this is a known risk pattern. If the quick-fix prompt omits Code Review (Step 7), it would allow unreviewed code to be committed.
- **Affected artifacts:** design-output.yaml file_inventory (quick-fix.prompt.md, 25 lines, "Simplified pipeline for trivial fixes"), design.md "Implementation Checklist" → Entry points
- **Recommendation:** Add an explicit design decision (D-13) defining the quick-fix prompt's step set. At minimum, it MUST include: Setup (Step 1), Code Review (Step 7), and Completion (Step 8) — the immutable minimum step set from the established D-14 principle. Document this as a constraint in both the design and the prompt's implementation spec.
- **Evidence:** design-output.yaml file_inventory lists `quick-fix.prompt.md` with `line_estimate: 25` and no step-set specification. design.md mentions it in "Implementation Order Recommendation" as "Entry points: feature-workflow.prompt.md, quick-fix.prompt.md" with no further detail. Repository memory confirms this exact pattern was a Critical code-review finding in a prior pipeline.

### Finding C-3: Feedback loop behavior with parallel DAG execution underspecified

- **Severity:** Minor
- **Description:** When the Tester returns `NEEDS_REVISION` during the Implementation-Testing loop (Step 5→6→5), the design doesn't specify behavior when multiple tasks were running in parallel in the preceding wave. Does the entire wave halt? Are completed tasks from the same wave preserved? Does the re-dispatched task see the state left by its parallel siblings? D-6 describes the happy-path DAG dispatch but not the failure-during-parallel-execution scenario.
- **Affected artifacts:** design-output.yaml pipeline_design Step 5 (max_cycles: 3), D-6 (DAG execution), design.md "Pipeline Design" → "Step 5: Implementation"
- **Recommendation:** Add to the Orchestrator's DAG dispatch description: "When a task returns NEEDS_REVISION during a parallel wave, all completed tasks in the same wave are marked as satisfied. Only the failed task is re-dispatched. The orchestrator does not roll back completed tasks."
- **Evidence:** design-output.yaml pipeline_design Step 5 says "All tasks DONE → Step 6 | NEEDS_REVISION → re-dispatch task" but doesn't address mixed results in a parallel wave. D-6 alt-1 describes dispatch logic but not failure handling during parallel execution.

---

## Summary

The design is well-structured and represents a significant security improvement over the current system — upgrading from instruction-only tool restrictions to VS Code platform-enforced `tools` frontmatter (D-3, D-4). The elimination of SQLite removes an entire class of injection risks. However, three residual gaps require attention: (1) instruction-only enforcement remains for `run_in_terminal` scope and `fetch_webpage` toggle — these should be documented as accepted risks with audit trail requirements; (2) the Knowledge agent's post-review execution with write tools creates an unreviewed mutation path that needs a pre-commit verification step; (3) the quick-fix prompt's step set is undefined, repeating a known pattern that previously required a Critical finding to catch. No Blocker issues found — all findings are addressable without architectural changes.
