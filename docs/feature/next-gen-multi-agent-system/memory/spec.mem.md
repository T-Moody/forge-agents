# Memory: spec

## Status

DONE: Feature specification produced with 5 architectural directions, 15 common requirements, 9 functional requirement groups, 7 non-functional requirements, 15 acceptance criteria, 10 edge cases, 3 user stories, 20 test scenarios, and full constraints/dependencies/risks sections.

## Key Findings

- Five architectural directions defined (Lean Pipeline 🟡, Forge-Plus 🟢, Anvil-Core 🔴, Hybrid Typed 🟡, Event-Driven 🔴) with sufficient detail for designer to select or synthesize an approach
- 15 common requirements apply to ALL directions, anchored by: three-state completion contracts, adversarial multi-model review (1 or 3 models by risk), evidence gating with machine-checkable gates, unified severity taxonomy (Blocker/Critical/Major/Minor), and risk classification (🟢/🟡/🔴)
- Key design decision deferred to designer: whether to use SQL-native state (Directions C/E), typed YAML schemas (Directions A/D), or hybrid Markdown+YAML (Direction B) for inter-agent communication
- All edge cases include fallback strategies: SQL→YAML fallback, multi-model→same-model fallback, partial pipeline→resumability
- Pushback system and structured multiple-choice approval mode specified as common requirements across all directions

## Highest Severity

N/A

## Decisions Made

- Deferred architectural direction selection to Design Agent (per user instruction: "Design agent should handle these decisions")
- Specified YAML as the fallback for SQL-dependent features (evidence ledger, state management) to avoid hard SQLite dependency
- Unified severity taxonomy to Blocker/Critical/Major/Minor (replacing Forge's inconsistent per-cluster vocabularies)
- Specified pipeline resumability as a requirement (EC-5) despite it not being present in either source system

## Artifact Index

- [feature.md](../feature.md)
  - §Title & Short Summary — one-line description and decision note about 5 directions
  - §Background & Context — source systems, research references, key research findings summary
  - §Architectural Directions — 5 complete directions (A: Lean Pipeline, B: Forge-Plus, C: Anvil-Core, D: Hybrid Typed, E: Event-Driven) each with agent list, pipeline structure, communication, memory, verification, adversarial review, approval mode, tradeoffs, risk rating
  - §Architectural Direction Comparison Matrix — 11-dimension comparison table across all 5 directions
  - §Common Requirements (CR-1 through CR-15) — requirements applying to ALL directions
  - §Functional Requirements (FR-1 through FR-9) — agent lifecycle, pipeline execution, state management, verification, adversarial review, memory, error handling, approval, output structure
  - §Non-Functional Requirements (NFR-1 through NFR-7) — determinism, auditability, scalability, robustness, context efficiency, maintainability, security
  - §Constraints & Assumptions — 8 constraints (C-1 through C-8), 7 assumptions (A-1 through A-7)
  - §Acceptance Criteria (AC-1 through AC-15) — testable criteria with pass/fail definitions
  - §Edge Cases & Error Handling (EC-1 through EC-10) — each with input/condition, expected behavior, severity
  - §User Stories / Flows (US-1 through US-3) — standard flow, high-risk flow, failure recovery flow
  - §Test Scenarios (TS-1 through TS-20) — each linked to acceptance criteria or edge cases
  - §Dependencies & Risks — dependency table with fallback impacts, risk matrix with mitigations
