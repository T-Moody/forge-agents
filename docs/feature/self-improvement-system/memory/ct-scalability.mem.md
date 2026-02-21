# Memory: ct-scalability

## Status

DONE: scalability re-review (Revision 1) — 7 findings (4 Medium, 3 Low). 2 previous High findings fully resolved (memory.md telemetry growth, telemetry recording overhead). Remaining concerns centered on evaluation file accumulation without run correlation, orchestrator telemetry recall accuracy from context, orchestrator definition size growth, and PostMortem ingestion volume across re-runs.

## Key Findings

- Evaluation files accumulate across re-runs with no run-correlation ID; PostMortem discovers ALL files via file_search and cannot distinguish current-run from prior-run data, producing misleading aggregate metrics (F5+F7, unchanged Medium)
- Orchestrator telemetry in "working context" depends on LLM accurately recalling dispatch metadata scattered across conversation history from Steps 1–7 when formatting PostMortem dispatch at Step 8; recall accuracy degrades with pipeline length (F10, new Medium)
- Orchestrator definition grows from 37.5 KB to estimated 42–44 KB (Track A only), compounds with telemetry recall pressure (F3, downgraded to Medium)
- PostMortem ingestion volume unbounded across re-runs despite memory-first reading; 5 re-runs × 34 evaluations = 170 files to grep (F2, downgraded to Medium)
- Telemetry data lost on failed runs — exactly when most diagnostically valuable; agent memories provide only binary status fallback (new Low)

## Highest Severity

Medium

## Decisions Made

(none — CT agents identify problems, not solutions)

## Artifact Index

- [ct-review/ct-scalability.md](../ct-review/ct-scalability.md)
  - §Previous Finding Disposition — table mapping 9 v0 findings to resolution status (2 resolved, 3 downgraded, 3 unchanged, 1 narrowed)
  - §Findings — 7 findings: evaluation file accumulation without run correlation (Medium), PostMortem ingestion volume unbounded across re-runs (Medium), orchestrator context window pressure from telemetry accumulation (Medium, new), orchestrator definition size growth (Medium), N+1 tool calls in PostMortem (Low), telemetry loss on failed runs (Low, new), Pattern C file amplification (Low)
  - §Cross-Cutting Observations — stale evaluation correctness (CT-Strategy), orchestrator size maintainability (CT-Maintainability), memory-first pattern as potential tech debt (CT-Maintainability)
  - §Requirement Coverage — 9 requirements assessed; FR-2.1, FR-2.5, FR-3.2, NFR-6, EC-7 improved; FR-4.3, EC-11 unchanged
