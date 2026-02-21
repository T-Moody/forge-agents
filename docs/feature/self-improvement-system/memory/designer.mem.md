# Memory: designer

## Status

DONE: Revised design (Revision 1) addressing CT cluster findings — 1 Critical, 6 High. Key changes: orchestrator keeps read tools / loses write tools only; telemetry via context passing not memory.md; shared evaluation schema; PostMortem memory-first reading with quantitative-only output; tool restriction as independent implementation task.

## Key Findings

- Telemetry moved from memory.md accumulation to orchestrator context passing — eliminates unbounded memory.md growth (~3.5-7KB/run never pruned), removes memory tool dependency for telemetry, eliminates compact telemetry DSL parsing contract
- Orchestrator write-tool restriction retains read tools (read_file, grep_search, semantic_search, file_search, list_dir, get_errors) — preserves cluster decision flows (CT, V, R) while preventing direct file modification; deviates from FR-6 literal text but satisfies intent
- Evaluation schema centralized in `.github/agents/evaluation-schema.md` — eliminates 14-file schema duplication, enables single-point schema evolution and versioning
- PostMortem produces quantitative metrics only (no improvement_recommendations) — clean boundary with R-Knowledge; uses memory-first reading pattern with targeted grep_search for context-efficient processing
- Tool restriction designed as independent implementation track (Track B) separable from self-improvement features (Track A) — reduces bundling risk, enables independent validation

## Highest Severity

N/A

## Decisions Made

- Telemetry via context passing not memory.md. Rationale: eliminates CT-3 bloat (3.5-7KB/run visible to all 19 agents), CT-7 memory tool dependency; ephemeral until PostMortem writes run-log is acceptable trade-off since agent memories persist completion statuses.
- Orchestrator retains read tools per CT-1 Critical resolution. Rationale: cluster decision flows require reading memory files; removing reads would break the pipeline's core coordination mechanism.
- Shared evaluation schema in `.github/agents/evaluation-schema.md`. Rationale: CT-4 identified 14-file duplication as High severity; single reference doc is standard practice; enables schema versioning.
- Removed improvement_recommendations from PostMortem report. Rationale: CT-6 identified R-Knowledge boundary blur; PostMortem is now purely quantitative (scores, frequencies, counts); R-Knowledge remains sole source of qualitative suggestions.
- Two-track implementation (Track A: self-improvement, Track B: write-tool restriction). Rationale: CT-2 identified bundling risk as High; independent tracks enable separate validation and rollback.
- Consistent sequence suffix collision avoidance across all file types. Rationale: CT-strategy F6 identified inconsistency between date-append and sequence-suffix strategies in v0 design.

## Artifact Index

- [design.md](../design.md)
  - §CT Revision Summary — table mapping 7 CT findings to design resolutions with section references
  - §High-level Architecture — revised data flow (context-based telemetry), revised PostMortem/R-Knowledge boundary (no recommendations), tool restriction as separate task (Track B)
  - §Data Models & DTOs — shared evaluation schema reference, context-passing telemetry structure (no compact DSL), revised PostMortem report schema (no improvement_recommendations), run-log format
  - §APIs & Interfaces — shared schema document spec, revised evaluating agent changes (reference doc not inline), revised orchestrator changes (write-tool restriction in Track B, context telemetry, Step 8 with telemetry dispatch), revised PostMortem agent (memory-first reading, quantitative only, targeted reads)
  - §Storage Architecture — revised append-only semantics (consistent sequence suffixes), no telemetry in memory.md, lazy directory creation
  - §Sequence / Interaction Notes — revised pipeline flow (telemetry in context not memory.md), revised telemetry data flow diagram
  - §Security Considerations — memory tool scope risk eliminated for telemetry; input validation for evaluation YAML; N/A for auth
  - §Failure & Recovery — 10 failure modes simplified (no memory.md telemetry failures), 5-level graceful degradation hierarchy
  - §Tradeoffs & Alternatives — 6 key decisions with CT-driven rationale (telemetry, schema, recommendations, tool restriction, reading pattern, retained decisions)
  - §Implementation Checklist — two independent tracks (A: self-improvement 7 deliverables, B: tool restriction 3 deliverables), 2 new files, 16 modified files (Track A) + 1 modified file (Track B), full AC mapping
