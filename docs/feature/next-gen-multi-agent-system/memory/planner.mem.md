# Memory: planner (v2 — fresh plan)

## Status

DONE: 13 tasks created across 4 execution waves. Plan covers all 14 output files (9 agent definitions, 3 reference documents, 1 prompt file, 1 README). Previous plan scrapped and replaced.

## Key Findings

- Wave 1 (foundation): schemas.md + dispatch-patterns/severity-taxonomy are critical path — all 8 non-orchestrator agents depend on them
- Wave 2 (8 agents): all non-orchestrator agents are independent of each other; execute as 2 sub-waves of ≤4 due to concurrency cap
- Orchestrator (Task 11) is the most complex single task: 5 responsibilities, 7 SQL queries, full decision table, DR-1 constraint — placed in Wave 3 after all other agents
- schemas.md (Task 01) must reconcile design.md dual schema definitions (§Decision 6 vs §Data Storage) — prefer §Decision 6 as canonical (includes v4 CHECK constraints)
- Only Task 13 (README) uses documentation-writer agent; all others use implementer

## Highest Severity

N/A

## Decisions Made

- **Grouped dispatch-patterns.md and severity-taxonomy.md into one task (Task 02):** Both are small reference documents, 2 files within 3-file limit — Confidence: High
- **8 non-orchestrator agents in Wave 2, orchestrator separate in Wave 3:** Orchestrator references all agents by name; placing it after Wave 2 ensures consistency. Other agents only depend on schemas.md and severity-taxonomy.md, not each other. — Confidence: High
- **feature-workflow.prompt.md (Task 12) in Wave 3 alongside orchestrator:** Simple binding file but logically pairs with orchestrator — Confidence: High
- **README in Wave 4 depends on Task 11 + Task 12:** README references all agents and the prompt; needs complete system picture — Confidence: High
- **documentation-writer for Task 13 only:** All other tasks produce agent definitions or reference docs requiring design.md understanding — implementer agent is more appropriate — Confidence: High

## Artifact Index

- [plan.md](../plan.md)
  - §Ordered Task Index — 13 tasks with file outputs, priority, agent assignment
  - §Execution Waves — 4 waves with dependency annotations
  - §Dependency Graph — Visual representation
  - §Implementation Specification — Code structure, source references, agent template, design constraints
  - §Plan Validation — Circular dependency check, task size validation, dependency existence check
  - §Pre-Mortem Analysis — Per-task failure scenarios, overall risk level, key assumptions
- [tasks/01-schemas-reference.md](../tasks/01-schemas-reference.md) — schemas.md: 10 YAML schemas + SQLite schemas + producer/consumer table
- [tasks/02-reference-documents.md](../tasks/02-reference-documents.md) — dispatch-patterns.md + severity-taxonomy.md
- [tasks/03-researcher-agent.md](../tasks/03-researcher-agent.md) — researcher.agent.md definition
- [tasks/04-spec-agent.md](../tasks/04-spec-agent.md) — spec.agent.md definition
- [tasks/05-designer-agent.md](../tasks/05-designer-agent.md) — designer.agent.md definition
- [tasks/06-planner-agent.md](../tasks/06-planner-agent.md) — planner.agent.md definition
- [tasks/07-implementer-agent.md](../tasks/07-implementer-agent.md) — implementer.agent.md definition
- [tasks/08-verifier-agent.md](../tasks/08-verifier-agent.md) — verifier.agent.md (most complex: 4-tier cascade + SQL)
- [tasks/09-adversarial-reviewer-agent.md](../tasks/09-adversarial-reviewer-agent.md) — adversarial-reviewer.agent.md definition
- [tasks/10-knowledge-agent.md](../tasks/10-knowledge-agent.md) — knowledge-agent.agent.md definition
- [tasks/11-orchestrator-agent.md](../tasks/11-orchestrator-agent.md) — orchestrator.agent.md (most complex: 5 responsibilities, decision table, SQL queries)
- [tasks/12-feature-workflow-prompt.md](../tasks/12-feature-workflow-prompt.md) — feature-workflow.prompt.md
- [tasks/13-readme-documentation.md](../tasks/13-readme-documentation.md) — README.md with Mermaid diagram
