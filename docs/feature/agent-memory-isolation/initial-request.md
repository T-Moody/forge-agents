# Initial Request: Agent-Isolated Memory System

## User Request

Update the agent prompt files in `NewAgentsAndPrompts/` to remove pipeline bottlenecks by redesigning the memory architecture. The core change is:

### 1. Agent-Isolated Memory (Branch-Merge Model)

Each subagent creates its own isolated memory file during execution, rather than writing to a shared `memory.md`. When the agent completes, the orchestrator merges that agent's memory into the main shared memory. This follows a branch/merge model:

- **Branch:** Subagent starts → creates its own memory file
- **Commit:** Subagent writes key findings/decisions to its isolated memory
- **Merge:** Orchestrator reads the agent's memory and merges into shared memory

### 2. Remove Aggregator Agents

Since each agent writes its own memory and the orchestrator handles merging, the dedicated aggregator agents are no longer needed:

- Remove `ct-aggregator` — orchestrator reads individual CT sub-agent memories + artifacts directly
- Remove `v-aggregator` — orchestrator reads individual V sub-agent memories + artifacts directly
- Remove `r-aggregator` — orchestrator reads individual R sub-agent memories + artifacts directly
- Remove researcher synthesis mode — orchestrator reads individual research memories + artifacts directly

### 3. Remove `analysis.md`

Instead of spawning a synthesis researcher to combine research docs into `analysis.md`, downstream agents read the individual research artifacts directly (`research/architecture.md`, `research/impact.md`, `research/dependencies.md`, `research/patterns.md`). The orchestrator reads research memories (not full artifacts) to track progress.

### 4. Keep Individual Cluster Files — No Combined Files

Instead of combining cluster outputs into single files:

- **CT cluster:** Keep individual `ct-review/ct-*.md` files. The orchestrator reads CT memories to determine if any found Critical/High issues → loop back to designer. No `design_critical_review.md` needed.
- **V cluster:** Keep individual `verification/v-*.md` files. The orchestrator reads V memories to determine pass/fail. No `verifier.md` needed.
- **R cluster:** Keep individual `review/r-*.md` files. The orchestrator reads R memories to determine result. No `review.md` needed.

### 5. Orchestrator Reads Memories, Not Full Artifacts

The orchestrator should read only the compact memory summaries to make routing/flow decisions, not the full artifact files. Full artifacts are passed to downstream agents by file path.

### 6. Artifacts Are Separate From Memories

The existing artifact files (e.g., `research/architecture.md`, `ct-review/ct-security.md`) are still created by each agent. The memory file is an additional compact output. Memories are NOT a replacement for artifacts — they are a lightweight index for the orchestrator.

## Key Constraints

- All existing agent files in `NewAgentsAndPrompts/` are in scope for modification
- The `ct-aggregator.agent.md`, `v-aggregator.agent.md`, `r-aggregator.agent.md` files should be deprecated/removed
- The researcher synthesis mode should be removed
- The `dispatch-patterns.md` should be updated to reflect the new architecture
- The `feature-workflow.prompt.md` should be updated
- The `orchestrator.agent.md` is the primary file requiring changes

## Files In Scope

All files under `NewAgentsAndPrompts/`:

- `orchestrator.agent.md` (major changes)
- `researcher.agent.md` (remove synthesis mode)
- `ct-aggregator.agent.md` (deprecate)
- `v-aggregator.agent.md` (deprecate)
- `r-aggregator.agent.md` (deprecate)
- `dispatch-patterns.md` (update patterns)
- `feature-workflow.prompt.md` (update)
- `spec.agent.md` (update inputs — no analysis.md)
- `designer.agent.md` (update inputs — no analysis.md)
- `planner.agent.md` (update inputs — no analysis.md, no design_critical_review.md)
- `implementer.agent.md` (add memory output)
- `documentation-writer.agent.md` (add memory output)
- `ct-security.agent.md`, `ct-scalability.agent.md`, `ct-maintainability.agent.md`, `ct-strategy.agent.md` (add memory output)
- `v-build.agent.md`, `v-tests.agent.md`, `v-tasks.agent.md`, `v-feature.agent.md` (add memory output)
- `r-quality.agent.md`, `r-security.agent.md`, `r-testing.agent.md`, `r-knowledge.agent.md` (add memory output)
- `critical-thinker.agent.md` (already deprecated — no changes needed)
