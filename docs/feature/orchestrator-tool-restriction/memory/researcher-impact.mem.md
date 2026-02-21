# Researcher Memory — Impact

- **Status:** DONE
- **Key Findings:**
  - Removing `grep_search`, `semantic_search`, `file_search`, `get_errors` has zero pipeline behavioral impact — orchestrator uses only known-path reads; these tools appear in 3 prompt locations (Global Rules §1, Operating Rules §1, §5) and Anti-Drift Anchor but are never used procedurally
  - Removing `memory.md` write access affects 18 distinct write operations: 10 merge points, 3 prune points, 3 self-verification logs, 1 initialization, and warning/invalidation writes across the Memory Lifecycle
  - All 19 subagents already have universal fallback: "If `memory.md` is missing, log a warning and proceed with direct artifact reads" — eliminating `memory.md` (Option B) activates existing graceful degradation
  - Cluster decision flows (CT, V, R) are fully read-only and work with just `read_file` on known `*.mem.md` paths — only the self-verification logging to `memory.md` breaks
  - R-Knowledge has the strongest dependency on aggregated Artifact Index from `memory.md`; all other agents use it as a convenience with robust fallback behavior
- **Highest Severity:** N/A
- **Decisions Made:** (none — research only)
- **Artifact Index:**
  - `research/impact.md` — §1 Tool Removal Impact (L1–L70), §2 Memory Write Removal Impact (L72–L165), §3 Cluster Decision Flows (L167–L205), §4 Wave Parsing and Task Dispatch (L207–L230), §5 Downstream Agent Impact (L232–L315)
