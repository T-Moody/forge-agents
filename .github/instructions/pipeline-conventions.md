# Pipeline Conventions

## Feature Directory Structure

Every pipeline feature creates a directory at `docs/feature/<feature-slug>/` containing:

- `spec-output.yaml` + `feature.md` — Specification
- `design-output.yaml` + `design.md` — Technical design
- `plan-output.yaml` + `plan.md` — Implementation plan
- `tasks/TASK-NNN.yaml` — Individual task definitions
- `implementation-reports/TASK-NNN.yaml` — Per-task implementation reports
- `verification-reports/TASK-NNN.yaml` — Per-task verification reports
- `review-findings/<scope>-<perspective>.md` — Review findings documents
- `review-verdicts/<scope>-<perspective>.yaml` — Review verdict summaries
- `verification-ledger.db` — SQLite database for evidence gates
- `evidence-bundle.md` — End-of-pipeline quality proof

## File Naming Patterns

- Agent outputs: `<artifact-type>-output.yaml` (e.g., `spec-output.yaml`, `design-output.yaml`)
- Task files: `TASK-NNN.yaml` (zero-padded 3-digit, e.g., `TASK-001`)
- Review verdicts: `<scope>-<perspective>.yaml` (e.g., `design-security-sentinel.yaml`, `code-architecture-guardian.yaml`)
- Review findings: `<scope>-<perspective>.md` (e.g., `design-security-sentinel.md`, `code-pragmatic-verifier.md`)

## Risk Classification

- 🟢 Standard: Well-understood change, <50 lines, single file
- 🟡 Elevated: Moderate complexity, 50-200 lines, affects 2+ consumers
- 🔴 Large: High complexity, >200 lines, core architectural change

## Pipeline Steps

0. Setup & Initialization
1. Research (×4 parallel)
   1a. Approval Gate
2. Specification
3. Design
   3b. Adversarial Design Review (×3 parallel)
4. Planning
   4a. Approval Gate
5. Implementation (wave-based, ≤4 concurrent)
6. Verification (per-task)
7. Adversarial Code Review (×3 parallel)
8. Knowledge Capture
9. Auto-Commit
