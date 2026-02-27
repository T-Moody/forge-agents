# Researcher Memory: Dependencies

- **Status:** DONE
- **Key Findings:**
  - Forge pipeline is an 8-step DAG with 23 agents, 3 dispatch patterns (A: parallel, B: sequential gate + parallel, C: replan loop), and file-based Markdown communication; orchestrator is the sole coordinator reading isolated `.mem.md` files for all routing decisions
  - Anvil is a single-agent 9-step internal loop using SQL (`anvil_checks` ledger + `session_store` recall) for verification tracking and evidence-based gating — no inter-agent file protocol; subagent dispatch limited to `code-review` with model selection
  - Memory systems are incompatible: Forge uses Markdown `.mem.md` files with Artifact Index for targeted reads + shared `memory.md`; Anvil uses SQL tables + VS Code `store_memory` — both share the `store_memory` cross-session mechanism but nothing else
  - Verification overlap is partial: Forge V-Build/V-Tests ≈ Anvil Tier 2, but Anvil adds IDE diagnostics (Tier 1), smoke tests (Tier 3), baseline capture, adversarial code review, and SQL-backed evidence bundles — all absent from Forge's V cluster
  - Neither system implements typed schemas at agent boundaries or MCP enforcement as recommended by the article; Forge's tool restrictions and completion contracts are prompt-level only; Anvil's `anvil_checks` SQL schema is the strongest typed contract in either system
- **Highest Severity:** N/A
- **Decisions Made:** (none — research only)
- **Artifact Index:**
  - [research/dependencies.md](../research/dependencies.md)
    - §Forge Agent Data Flow Graph — complete pipeline chain, read/write matrix, memory dependencies, coupling analysis, critical path
    - §Anvil Internal Data Flow — phase chain, verification cascade dependencies, external dependencies table
    - §Inter-System Dependencies for Merging — communication protocol comparison, memory overlap, verification overlap, CT vs adversarial review, data format compatibility
    - §External Dependencies — VS Code APIs, git operations, model/LLM requirements, tool requirements per agent category, MCP tools
    - §Article Design Principles as Dependency Constraints — typed schemas gap, action schemas gap, MCP enforcement gap, distributed systems gap analysis
