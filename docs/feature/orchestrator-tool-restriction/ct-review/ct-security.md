# Critical Review: Security & Backwards Compatibility (Iteration 3)

**Revision context:** Iteration 2 design revision corrected all 5 pre-existing "via `memory` tool" references to use "via subagent delegation" instead (design §3.5–§3.9). Phase 1 scope unchanged: tool restriction + memory tool disambiguation in 2 files, memory.md architecture preserved.

## Prior Findings Disposition

| Prior Finding | Prior Severity | Iteration 3 Status | Rationale |
|---|---|---|---|
| Prompt injection via Lessons Learned keyword extraction | High | **Resolved (iter 1)** | Keyword extraction heuristic removed (design §13.2). Existing memory.md Lessons Learned preserved. |
| Advisory-only tool enforcement | High → Low | **Carried at Low** | Removed tools are all read-only. Triple enforcement (YAML + prose + Anti-Drift) is strongest available. See Finding 1. |
| Audit trail loss on pipeline crash | High | **Resolved (iter 1)** | Memory.md preserved. Self-verification logging preserved. |
| VS Code memory tool conflation / disambiguation contradiction | Medium (iter 2) | **Resolved** | All 5 conflating "via `memory` tool" references corrected to subagent delegation (§3.5–§3.9). 3 disambiguation statements no longer contradict surrounding text. FR-6.2 now satisfied within Phase 1. See §Verification below. |
| Invalidation via dispatch prompt weaker than file markers | Medium | **Resolved (iter 1)** | File-based `[INVALIDATED]` markers preserved. |
| Subagent memory files not integrity-checked | Medium | **Resolved (iter 1)** | Memory.md merged-and-curated layer preserved. |
| Partial update leaves dangling memory.md references | Low | **Resolved (iter 1)** | Only 2 files change. No partial-update risk. |
| Stale path reference (dispatch-patterns.md) | Low | **Carried — out of scope** | Pre-existing issue unrelated to tool restriction. See Finding 2. |

## Verification: Memory Tool Disambiguation Fix

The iteration 2 design revision (§3.5–§3.9) specifies corrections for all 5 locations that previously conflated the VS Code `memory` tool with pipeline file writes:

| Location | Old (conflating) | New (corrected) | Design Section |
|---|---|---|---|
| Global Rule 12 (L44) | "via `memory` tool merging" | "by dispatching a subagent to perform the merge" | §3.5 |
| Global Rule 12 (L46) | "merges into `memory.md` (via `memory` tool)" | "dispatches a subagent to merge them into `memory.md`" | §3.5 |
| Step 0.1 (L188) | "Use the `memory` tool to create `memory.md`" | "Delegate to a setup subagent via `runSubagent` to create `memory.md`" | §3.6 |
| Step 8.2 (L456) | "using `memory` tool or delegating to subagent" | "dispatches a subagent to merge" | §3.7 |
| Memory Lifecycle Table (L503) | "Use `memory` tool to create `memory.md`" | "Delegate to a setup subagent to create `memory.md`" | §3.8 |

Additionally, §3.9 corrects 8 merge steps (1.1m, 2m, 3m, 3b.2, 4m, between-waves, 6.3, 7.3) from "orchestrator merges" to "orchestrator dispatches a subagent to merge" and removes the incorrect Step 1.1m assertion "This is an orchestrator merge operation — no subagent invocation."

**Assessment:** The contradiction identified in iteration 2 is fully resolved in the design. Post-implementation, the orchestrator prompt will contain:
- 3 disambiguation statements (Global Rule 1, Operating Rule 5, Anti-Drift Anchor) asserting `memory` tool is cross-session only
- 0 references conflating `memory` tool with pipeline file writes
- Consistent "subagent delegation" language for all memory.md write operations

Test T-13 ("Zero matches for 'via `memory` tool' in orchestrator.agent.md") provides a concrete verification gate.

## Findings

### [Severity: Low] Advisory-Only Tool Enforcement — Accepted Residual Risk

- **What:** The YAML `tools:` field and prose restrictions are advisory-only with no runtime enforcement, detection, or post-hoc audit. All 4 removed tools (`grep_search`, `semantic_search`, `file_search`, `get_errors`) are read-only. Unauthorized use results in unnecessary codebase exploration, not data corruption or state modification. The design's triple enforcement (YAML frontmatter + Operating Rule 5 + Anti-Drift Anchor) is the strongest mechanism available and is consistent with how all 20 other agents enforce tool boundaries.
- **Where:** Design §2.4 (YAML convention), §3.3 (Operating Rule 5), §3.4 (Anti-Drift Anchor), §8.3 (Threat Model), §13.3 (Tradeoff)
- **Likelihood:** Low — the orchestrator prompt is long (~546 lines, increasing drift probability), but the removed tools are read-only and the orchestrator's pipeline logic already uses only `read_file` on known paths
- **Impact:** Low — worst case is ad-hoc exploration (wasting tokens). No data integrity risk, no state corruption, no privilege escalation.
- **Assumption at risk:** That triple-layered prompt instructions are sufficient to prevent read-only tool misuse.

### [Severity: Low] Stale Dispatch Patterns Path Reference — Pre-existing

- **What:** The orchestrator prompt (L96) references `NewAgentsAndPrompts/dispatch-patterns.md` but the actual file path is `.github/agents/dispatch-patterns.md`. Pre-existing, out of scope (design §14.2).
- **Where:** orchestrator.agent.md L96
- **Likelihood:** High — already broken
- **Impact:** Low — dispatch patterns are summarized inline in the orchestrator prompt
- **Assumption at risk:** That the orchestrator can resolve the stale path.

## Cross-Cutting Observations

- **CT-Strategy scope:** The feature spec (feature.md) was written for the full scope (23 files, 15 ACs). Phase 1 implements a subset (2 files, 7 ACs). The design §16.3 maps coverage and §16.5 provides guidance for verifiers. This is a spec-maintenance concern, not a security gap.
- **CT-Maintainability scope:** The expanded edit count (~18 edits vs. original ~5) increases implementation error surface within orchestrator.agent.md. All edits are well-specified with exact current/new text in the design, which mitigates but does not eliminate copy-paste error risk.

## Requirement Coverage

| Requirement | Coverage Status | Notes |
| ----------- | --------------- | ----- |
| FR-1 (Tool Restriction) | Fully covered | Triple enforcement for read-only tools. No destructive risk. |
| FR-6.1 (Memory Tool Disambiguation — add clarity) | Fully covered | 3 disambiguation points added in Global Rule 1, Operating Rule 5, Anti-Drift Anchor. |
| FR-6.2 (Remove conflating memory tool language) | **Fully covered** | All 5 conflating "via `memory` tool" references corrected to subagent delegation (§3.5–§3.9). Iteration 2 tension between FR-6.2 and memory.md preservation is resolved. |
| NFR-1 (Backward Compatibility) | Fully covered | Only 2 files change. Memory.md preserved. All subagents untouched. Zero backward compatibility risk. |
| NFR-4.1 (YAML/prose consistency) | Fully covered | Design §3.3 specifies identical tool lists in YAML and Operating Rule 5. |
| NFR-5 (Cluster Decision Integrity) | Fully covered | Cluster decision flows unchanged. Memory.md persistent logging preserved. |
| AC-3 (Removed tool refs eliminated) | Fully covered | All 4 locations containing removed tool names are modified. |
| AC-11 (Memory tool disambiguated) | Fully covered | 3 disambiguation statements + 5 contradicting references corrected + 8 merge step wordings fixed. Zero conflating language remains. |
| EC-7 (Memory tool conflation) | Fully covered | Phase 1 both adds disambiguation AND removes all contradictions. Net improvement over pre-existing state. |
