# Knowledge Evolution Analysis

## Instruction Suggestions

### [Category: instruction-update] Residual "merges" Language in Summary-Level References

- **File:** `.github/agents/orchestrator.agent.md`
- **Change:** Update
- **Rationale:** CT-Maintainability (iter 3, Low) identified 3 summary-level references that still use bare "merges" language without specifying subagent delegation. Phase 1 corrected all 8 merge step bodies and all 5 "via `memory` tool" references, but left summary-level language untouched (by design — scope control). These residual "merges" references imply direct orchestrator file action, inconsistent with the corrected detailed wording ("dispatches a subagent to merge"). While the LLM will likely resolve the ambiguity from adjacent context, aligning them would eliminate the inconsistency entirely.
- **Before:**
  - Global Rule 6 (L39): "orchestrator reads the agent's `memory/<agent>.mem.md` and merges Key Findings..."
  - Memory Lifecycle Merge row (L503): "Orchestrator reads `memory/<agent>.mem.md`, merges Key Findings..."
  - Memory Lifecycle post-mortem row (~L510): "Read `memory/post-mortem.mem.md`, merge into `memory.md`"
- **After:**
  - Global Rule 6: "orchestrator reads the agent's `memory/<agent>.mem.md` and dispatches a subagent to merge Key Findings..."
  - Memory Lifecycle Merge row: "Orchestrator reads `memory/<agent>.mem.md`, dispatches a subagent to merge Key Findings..."
  - Memory Lifecycle post-mortem row: "Read `memory/post-mortem.mem.md`, dispatch a subagent to merge into `memory.md`"

### [Category: instruction-update] Stale Dispatch Patterns Path Reference

- **File:** `.github/agents/orchestrator.agent.md`
- **Change:** Update
- **Rationale:** CT-Security (iter 3, Low) identified that L97 references `NewAgentsAndPrompts/dispatch-patterns.md` but the actual file is `.github/agents/dispatch-patterns.md`. This is a pre-existing bug — the path was never updated when the project migrated from the `NewAgentsAndPrompts/` directory structure to `.github/agents/`. The orchestrator's inline summaries of Pattern A and B mean this broken reference has low impact, but it should be corrected for accuracy.
- **Before:** `> Patterns A and B full definitions: \`NewAgentsAndPrompts/dispatch-patterns.md\`.`
- **After:** `> Patterns A and B full definitions: \`.github/agents/dispatch-patterns.md\`.`

### [Category: instruction-update] Orchestrator Expectations Table Uses Bare "merges" Language

- **File:** `.github/agents/orchestrator.agent.md`
- **Change:** Update
- **Rationale:** The Orchestrator Expectations table (L477–L492) has 8 rows that say "Orchestrator reads memory and merges" without mentioning subagent delegation. Same root cause as the residual "merges" language above. These rows are summary-level and lower severity (table cell brevity is expected), but they create another location where implied direct file action contradicts the corrected merge step wording.
- **Before:** "Orchestrator reads memory and merges" (8 rows)
- **After:** "Orchestrator reads memory and dispatches merge" (8 rows)

## Skill Suggestions

### [Category: skill-update] Implementer Text-Match Verification Pattern

- **Agent:** implementer
- **Change:** Add (documentation)
- **Rationale:** All 4 implementers in this feature used a consistent pattern: read the target file section first, match against the design's "Current" text to confirm no drift, then apply the edit. This was prompted by the design's own guidance ("matches on text content, not line numbers") but is not codified in the implementer agent's Operating Rules. For prompt-only change tasks where line numbers may drift, this pattern prevents misapplied edits.
- **Before:** No explicit guidance on text-match verification before edits.
- **After:** Operating Rule guidance: "For prompt/documentation edits, read the target section first and verify the 'Current' text matches before applying changes. Match on text content, not line numbers."

## Pattern Captures

### [Category: pattern-capture] Triple-Layered Tool Enforcement

- **Pattern:** Tool restrictions are enforced via three complementary mechanisms: (1) YAML `tools:` frontmatter declaration (machine-readable, advisory), (2) prose Operating Rule with explicit allowed/prohibited lists (primary enforcement), (3) Anti-Drift Anchor with prohibition list and rationale (reinforcement at end-of-prompt). Each layer serves a different purpose: YAML for tooling/scanning, prose for LLM instruction, Anti-Drift for drift resistance.
- **Evidence:** `.github/agents/orchestrator.agent.md` L4 (YAML), L93 (Operating Rule 5), L540 (Anti-Drift Anchor). Design §2.4 documents the rationale.
- **Applicability:** Any agent that needs formal tool restrictions. The orchestrator is the first; future high-risk agents (e.g., agents with access to `run_in_terminal`) could adopt the same triple-layer pattern.

### [Category: pattern-capture] Memory Tool / Pipeline Concept Name Conflation Anti-Pattern

- **Pattern:** When a VS Code built-in tool shares a name with a pipeline concept (e.g., `memory` tool vs. `memory.md` file), prompt authors will conflate them. This happened 5 times in the orchestrator prompt where "via `memory` tool" was used to describe pipeline file writes — an operation the `memory` tool cannot perform. The fix is explicit disambiguation statements in multiple locations plus elimination of all conflating references.
- **Evidence:** Design §3.5–§3.9 documents the 5 conflating references. CT-Maintainability iter 2 rated this High. Phase 1 corrected all 5.
- **Applicability:** Any time a new tool or concept is introduced that shares a name with an existing concept. Preventive measure: add a disambiguation statement in the Operating Rules when the name overlap is first introduced.

### [Category: pattern-capture] Phase-Gating via CT Review

- **Pattern:** CT review identified scope coupling between two logically independent concerns (tool restriction + memory architecture redesign) and recommended splitting them into separate phases. This prevented a high-risk multi-concern feature from shipping as one monolithic change. The phasing strategy was: Phase 1 = low-risk restriction-only changes to 2 files, Phase 2 = high-risk architectural redesign (deferred to a future feature).
- **Evidence:** CT-Strategy iter 1 Finding 1 (High: scope coupling). Design §14 (Phase 2 deferral). Plan.md limits to 5 ACs.
- **Applicability:** Any feature where CT review reveals that the design bundles independent concerns. The pattern is: split into phases where Phase 1 delivers independent value without depending on Phase 2.

### [Category: pattern-capture] Pre-Existing Bug Fix Framing

- **Pattern:** When a feature change exposes pre-existing bugs in the same file, frame the bug fixes as corrections of the pre-existing state rather than behavioral changes of the new feature. This allows the fixes to be included in the same feature scope without expanding the feature's perceived risk profile. Design §3.5 Rationale explicitly calls the "via `memory` tool" language a "pre-existing bug."
- **Evidence:** Design §1.3 ("The existing prompt was broken: it instructed an operation that was impossible"). CT-Maintainability cross-cutting observation ("strategically sound — allows corrections to be framed as bug fixes").
- **Applicability:** When a feature touches a file and the implementer discovers prompt instructions that describe impossible operations or reference non-existent capabilities.

### [Category: pattern-capture] Same-File Sequencing in Wave Planning

- **Pattern:** When multiple tasks edit the same file, they must be sequenced into different waves even if logically independent. This prevents concurrent-edit conflicts where Task B's `oldString` no longer matches because Task A modified adjacent lines. The planner explicitly created a 2-wave structure (01→02) for this reason.
- **Evidence:** Plan.md §Execution Waves, §Dependency Graph. Tasks 01 and 02 both edit `orchestrator.agent.md`.
- **Applicability:** Any plan where multiple implementation tasks target the same file. The pattern generalizes: same-file tasks form a sequential chain regardless of logical independence.

## Decision Log Entries

### [2026-02-20] Orchestrator Tool Restriction — Phase 1 Scope

- **Context:** The initial request covered both tool restriction (removing 4 discovery tools) and memory architecture redesign (eliminating shared `memory.md`). CT-Strategy review (iter 1) identified scope coupling as High risk — both concerns modified the same file but had independent value propositions and different risk profiles.
- **Decision:** Split into Phase 1 (tool restriction only, 2 files, 5 ACs) and Phase 2 (memory architecture redesign, deferred to future feature). Phase 1 preserves `memory.md` entirely.
- **Rationale:** Tool restriction delivers immediate value (formalizing existing behavior) with near-zero risk. Memory elimination is a high-risk architectural change requiring new subagent patterns. Coupling them means a Phase 2 regression could block the Phase 1 benefit. Splitting allows Phase 1 to ship independently.
- **Scope:** Project-wide (establishes the orchestrator's formal tool boundary).
- **Affected Components:** `.github/agents/orchestrator.agent.md`, `.github/prompts/feature-workflow.prompt.md`, `docs/feature/orchestrator-tool-restriction/feature.md`.

### [2026-02-20] YAML `tools:` Frontmatter Convention Established

- **Context:** The orchestrator needed a machine-readable declaration of its allowed tools. No agent in the repository had previously used a `tools:` field in YAML frontmatter.
- **Decision:** Add `tools: [agent, agent/runSubagent, memory, read_file, list_dir]` to the orchestrator's YAML frontmatter. This establishes the `tools:` field as a convention for future agents.
- **Rationale:** The YAML field provides machine-readable declaration visible without reading the full prompt. VS Code runtime enforcement is advisory-only (may not be enforced), but the field serves as documentation and potential future enforcement hook. Triple-layer enforcement (YAML + prose + Anti-Drift) compensates for advisory nature.
- **Scope:** Project-wide (establishes YAML `tools:` convention).
- **Affected Components:** `.github/agents/orchestrator.agent.md` (YAML frontmatter), design.md §2.4 (convention documentation).

### [2026-02-20] Memory Tool Disambiguation as Standard Practice

- **Context:** The VS Code `memory` tool (cross-session knowledge store) shares the word "memory" with pipeline `memory.md` (operational memory file). This conflation produced 5 incorrect instructions in the orchestrator prompt that told the LLM to write pipeline files via a tool incapable of doing so.
- **Decision:** Add explicit disambiguation statements wherever the `memory` tool is referenced in tool lists, and replace all conflating references with correct subagent delegation language.
- **Rationale:** The conflation was a pre-existing bug that went undetected because the LLM likely ignored the impossible instruction and fell back to subagent delegation. Making the distinction explicit prevents future prompt authors from reintroducing the conflation. Three disambiguation points (Global Rule 1, Operating Rule 5, Anti-Drift Anchor) provide redundant clarity.
- **Scope:** Project-wide (any agent that references both the `memory` tool and pipeline memory files).
- **Affected Components:** `.github/agents/orchestrator.agent.md` (6 sections modified).

## Summary

- **Total suggestions:** 7
- **Instruction updates:** 3
- **Skill updates:** 1
- **Pattern captures:** 5
- **Workflow improvements:** 0
- **Rejected (safety filter):** 0
