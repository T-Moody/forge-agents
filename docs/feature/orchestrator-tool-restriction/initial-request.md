# Initial Request: Orchestrator Tool Restriction

## Feature Request

Redesign the orchestrator agent's tool access and memory architecture.

### Problem Statement

The orchestrator currently has broad tool access including `read_file`, `grep_search`, `semantic_search`, `file_search`, `list_dir`, and `get_errors`. This allows it to do open-ended exploration and direct file manipulation beyond pure orchestration. Additionally, the orchestrator is the sole writer to a shared `memory.md` file, but if its write tools are restricted, the memory architecture needs rethinking.

### Goals

1. **Restrict orchestrator tools to Option B:** `tools: [agent, agent/runSubagent, memory, read_file, list_dir]`
   - Remove `grep_search`, `semantic_search`, `file_search`, `get_errors`
   - The orchestrator can read known paths and list directories, but cannot search/grep/discover
   - All writes remain delegated to subagents
   - `memory` here is VS Code's built-in cross-session memory tool, NOT pipeline memory.md

2. **Redesign the memory architecture** given the orchestrator can read but not write:
   - Option A: Keep shared `memory.md` but delegate all writes to a subagent
   - Option B: Remove shared `memory.md` entirely — orchestrator reads agent-isolated `memory/<agent>.mem.md` files directly via `read_file`
   - Option C: Hybrid — keep `memory.md` but have it auto-assembled by a dedicated memory-merge subagent
   - Determine the best approach

3. **Update all orchestrator prompt sections** that reference removed tools or direct file writing:
   - Global Rules
   - Operating Rules
   - Cluster Decision Logic (currently says "orchestrator reads memory files" — this stays with Option B tools)
   - Memory Lifecycle Actions
   - Anti-Drift Anchor
   - feature-workflow.prompt.md

4. **Ensure the pipeline still works** — cluster decisions, wave parsing, task dispatch, memory merging all must function correctly with the restricted tool set.

### Key Constraints

- This is an additive/modified change to existing agent definitions
- Must not break existing pipeline functionality
- The orchestrator's `read_file` access to `memory/<agent>.mem.md` files is essential for cluster routing decisions
- All file creation/modification must go through subagents
- The VS Code `memory` tool (for cross-session codebase knowledge) is separate from pipeline `memory.md`

### Context

- Previous analysis (from designer agent) recommended Option B tools as the best balance of purity, efficiency, and reliability
- The key design question remaining is the memory architecture: what happens to shared `memory.md` when the orchestrator can't write to it?
- Current files to modify: `.github/agents/orchestrator.agent.md`, `.github/prompts/feature-workflow.prompt.md`
