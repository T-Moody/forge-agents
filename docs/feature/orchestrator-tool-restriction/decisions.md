# Architectural Decision Log

## [2026-02-20] Orchestrator Tool Restriction — Phase 1 Scope

- **Context:** The initial request covered both tool restriction (removing 4 discovery tools) and memory architecture redesign (eliminating shared `memory.md`). CT-Strategy review (iter 1) identified scope coupling as High risk — both concerns modified the same file but had independent value propositions and different risk profiles.
- **Decision:** Split into Phase 1 (tool restriction only, 2 files, 5 ACs) and Phase 2 (memory architecture redesign, deferred to future feature). Phase 1 preserves `memory.md` entirely.
- **Rationale:** Tool restriction delivers immediate value (formalizing existing behavior) with near-zero risk. Memory elimination is a high-risk architectural change requiring new subagent patterns. Coupling them means a Phase 2 regression could block the Phase 1 benefit. Splitting allows Phase 1 to ship independently.
- **Scope:** Project-wide (establishes the orchestrator's formal tool boundary).
- **Affected components:** `.github/agents/orchestrator.agent.md`, `.github/prompts/feature-workflow.prompt.md`, `docs/feature/orchestrator-tool-restriction/feature.md`.

## [2026-02-20] YAML `tools:` Frontmatter Convention Established

- **Context:** The orchestrator needed a machine-readable declaration of its allowed tools. No agent in the repository had previously used a `tools:` field in YAML frontmatter.
- **Decision:** Add `tools: [agent, agent/runSubagent, memory, read_file, list_dir]` to the orchestrator's YAML frontmatter. This establishes the `tools:` field as a convention for future agents.
- **Rationale:** The YAML field provides machine-readable declaration visible without reading the full prompt. VS Code runtime enforcement is advisory-only (may not be enforced), but the field serves as documentation and potential future enforcement hook. Triple-layer enforcement (YAML + prose + Anti-Drift) compensates for advisory nature.
- **Scope:** Project-wide (establishes YAML `tools:` convention).
- **Affected components:** `.github/agents/orchestrator.agent.md` (YAML frontmatter), `docs/feature/orchestrator-tool-restriction/design.md` §2.4 (convention documentation).

## [2026-02-20] Memory Tool Disambiguation as Standard Practice

- **Context:** The VS Code `memory` tool (cross-session knowledge store) shares the word "memory" with pipeline `memory.md` (operational memory file). This conflation produced 5 incorrect instructions in the orchestrator prompt that told the LLM to write pipeline files via a tool incapable of doing so.
- **Decision:** Add explicit disambiguation statements wherever the `memory` tool is referenced in tool lists, and replace all conflating references with correct subagent delegation language.
- **Rationale:** The conflation was a pre-existing bug that went undetected because the LLM likely ignored the impossible instruction and fell back to subagent delegation. Making the distinction explicit prevents future prompt authors from reintroducing the conflation. Three disambiguation points (Global Rule 1, Operating Rule 5, Anti-Drift Anchor) provide redundant clarity.
- **Scope:** Project-wide (any agent that references both the `memory` tool and pipeline memory files).
- **Affected components:** `.github/agents/orchestrator.agent.md` (6 sections modified).
