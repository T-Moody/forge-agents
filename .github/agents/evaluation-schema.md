# Artifact Evaluation System

> Tier 2 shared reference. SQLite-based evaluation replaces Forge's per-file YAML approach (DP-3).

## §1 Overview

Every agent that consumes an upstream artifact evaluates it after use. Evaluations are SQL INSERT operations into the `artifact_evaluations` table in `verification-ledger.db`, queryable by the Knowledge Agent for pipeline quality analysis.

## §2 Table Schema Reference

The `artifact_evaluations` table is defined in [schemas.md](schemas.md) under **SQLite Schemas**. Key columns:

| Column                | Type    | Description                                                 |
| --------------------- | ------- | ----------------------------------------------------------- |
| `evaluator_agent`     | TEXT    | Agent performing the evaluation                             |
| `evaluator_instance`  | TEXT    | Instance identifier (nullable)                              |
| `artifact_path`       | TEXT    | Relative path to the evaluated artifact                     |
| `usefulness_score`    | INTEGER | 1–10 rating (see §4 rubric)                                 |
| `clarity_score`       | INTEGER | 1–10 rating (see §4 rubric)                                 |
| `missing_information` | TEXT    | What was expected but absent (≤ 2000 chars)                 |
| `inaccuracies`        | TEXT    | What was wrong or misleading (≤ 2000 chars)                 |
| `impact_on_work`      | TEXT    | How artifact quality affected task execution (≤ 2000 chars) |
| `run_id`              | TEXT    | Pipeline run identifier                                     |

## §3 Evaluation Workflow

1. **Read** the upstream artifact as part of your normal workflow
2. **Use** the artifact to complete your task
3. **Evaluate** the artifact's quality based on your experience using it
4. **INSERT** the evaluation via `run_in_terminal` using the template in sql-templates.md §4

Evaluate **after** you have used the artifact, not before — your assessment must reflect actual utility.

## §4 Scoring Guidelines

### Usefulness Score (1–10)

| Range | Label                | Description                                                                   |
| ----- | -------------------- | ----------------------------------------------------------------------------- |
| 1–3   | Harmful / Misleading | Actively harmful or misleading; required significant rework or led you astray |
| 4–5   | Below Expectations   | Missing key information or unclear; needed substantial supplementation        |
| 6–7   | Adequate             | Usable with minor gaps; got the job done with some extra effort               |
| 8–9   | Good                 | Clear, complete, and actionable; minor improvements possible                  |
| 10    | Exceptional          | No improvements needed; everything required was present and correct           |

### Clarity Score (1–10)

| Range | Label            | Description                                                             |
| ----- | ---------------- | ----------------------------------------------------------------------- |
| 1–3   | Incomprehensible | Confusing structure, contradictory statements, or unreadable formatting |
| 4–5   | Unclear          | Understandable with effort; ambiguous sections or poor organization     |
| 6–7   | Adequate         | Generally clear; minor formatting or structural issues                  |
| 8–9   | Clear            | Well-structured, unambiguous, easy to follow                            |
| 10    | Crystal Clear    | Perfect structure, zero ambiguity, exemplary formatting                 |

## §5 Structured Text Fields

- **`missing_information`**: Describe what you needed but the artifact did not provide. Be specific — name the section, field, or data point that was absent.
- **`inaccuracies`**: Describe what was wrong or misleading. Reference specific claims that did not match reality.
- **`impact_on_work`**: Describe how the artifact's quality affected your task. Did gaps cause rework? Did clarity issues cause misinterpretation?

Leave fields as empty string (`''`) if not applicable — do not use NULL for "nothing to report."

## §6 Phase 1 Agents (Required)

| Agent                    | Evaluates                                                              | Artifact Example                                                                           |
| ------------------------ | ---------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| **Implementer**          | Task definitions                                                       | `tasks/TASK-NNN.yaml` — Was the task clear? Was `relevant_context` sufficient?             |
| **Verifier**             | Implementation reports                                                 | `implementation-reports/TASK-NNN.yaml` — Was the report accurate? Were changes documented? |
| **Adversarial Reviewer** | Design output (Step 3b) or implementation + verification data (Step 7) | `design-output.yaml`, implementation files                                                 |

Phase 1 agents **must** INSERT an evaluation for every upstream artifact they consume.

## §7 Phase 2 Expansion (Future)

| Agent    | Evaluates                            |
| -------- | ------------------------------------ |
| Spec     | Research outputs (`research/*.yaml`) |
| Designer | Spec output (`spec-output.yaml`)     |
| Planner  | Design output (`design-output.yaml`) |

Phase 2 agents will be onboarded after Phase 1 evaluation quality is validated by the Knowledge Agent.

**Consumption**: The Knowledge Agent reads `artifact_evaluations` via SQL queries to produce quantitative summaries (mean scores, worst-rated artifacts, common gaps) in `knowledge-output.yaml`.

## §8 E2E Interaction Log Quality

Evaluates the completeness and determinism of E2E interaction logs (per D-18 determinism model).

| Score | Label    | Criteria                                                                                                                                            |
| ----- | -------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| 0     | Missing  | `interaction_log` absent or empty for skills that require it                                                                                        |
| 1     | Partial  | `interaction_log` present but incomplete — missing fields (`action`, `target`, `result`, or `evidence_path`) or steps out of skill-definition order |
| 2     | Complete | All steps present with `action`, `target`, `result`, `evidence_path`, and `duration_ms`; step order matches skill definition exactly (FR-14.1)      |

**Required fields per step:** `action`, `target`, `result`, `evidence_path`, `duration_ms`.

## §9 Evidence Completeness

Evaluates the presence, integrity, and safety of E2E evidence artifacts.

| Score | Label      | Criteria                                                                                                                               |
| ----- | ---------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| 0     | Missing    | `evidence-manifest.yaml` absent                                                                                                        |
| 1     | Incomplete | `evidence-manifest.yaml` present but: SHA-256 hashes missing, referenced files not on disk, or sanitization not verified               |
| 2     | Verified   | Manifest present with SHA-256 hashes, all referenced evidence files exist on disk, no credential/token leakage (sanitization verified) |

**Checklist:** manifest presence → SHA-256 hash per entry → file existence → credential sanitization.

## §10 TDD Compliance Quality

Evaluates adherence to the RED → GREEN → VERIFY TDD cycle.

| Score | Label   | Criteria                                                                                                                                                                                              |
| ----- | ------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 0     | None    | No TDD evidence — `tdd_red_green` section missing or `tests_written_first` is false with no justification                                                                                             |
| 1     | Partial | Partial cycle — e.g., failing test evidenced (RED) but implementation is speculative (GREEN violated), or structured `check_name` results absent in SQL (VERIFY missing)                              |
| 2     | Full    | Complete RED → GREEN → VERIFY cycle: failing test committed before implementation, minimal production code (no speculative additions), and structured `check_name` results recorded in `anvil_checks` |

**Phases:** RED (failing test before code) → GREEN (minimal implementation) → VERIFY (structured SQL check results).
