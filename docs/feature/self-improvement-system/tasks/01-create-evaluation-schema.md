# Task 01: Create Shared Evaluation Schema Document

**Task Goal:** Create `.github/agents/evaluation-schema.md` — the single shared reference document defining the artifact evaluation YAML schema used by all 14 evaluating agents and the PostMortem agent.

**depends_on:** none

**agent:** implementer

---

## In-Scope

- Create new file `.github/agents/evaluation-schema.md`
- Include the full `artifact_evaluation` YAML schema with field descriptions and constraints
- Include the `evaluation_error` fallback schema
- Include rules for list fields, score ranges, non-blocking behavior, and collision avoidance
- Include a version identifier for future schema evolution

## Out-of-Scope

- Modifying any existing agent files (that's Tasks 05–09)
- Defining the PostMortem report schema (that's Task 02)
- Any runtime validation logic (this is a reference document, not executable code)

---

## Acceptance Criteria

1. File exists at `.github/agents/evaluation-schema.md`
2. Contains the `artifact_evaluation` YAML schema with all 9 fields: `evaluator`, `source_artifact`, `usefulness_score`, `clarity_score`, `useful_elements`, `missing_information`, `information_not_used`, `inaccuracies`, `impact_on_work`
3. Field type constraints are documented: scores are integer 1–10, list fields must have ≥1 entry, use `"N/A — none identified"` if nothing applies
4. Contains the `evaluation_error` fallback schema with fields: `evaluator`, `source_artifact`, `error`
5. Contains rules section: one YAML block per source artifact, non-blocking behavior (evaluation failure ≠ ERROR), evaluation is secondary to primary output
6. Contains collision avoidance rule: if evaluation file already exists, append sequence suffix (`-2`, `-3`)
7. Contains a version identifier (e.g., `v1`)
8. YAML blocks are well-formed examples within fenced code blocks

---

## Estimated Effort

Low (~80 lines)

---

## Test Requirements (TDD Fallback — Configuration-Only)

TDD skipped: documentation-only task

Since this is a Markdown reference document with no test framework:

1. **Verify file exists** at `.github/agents/evaluation-schema.md` ✅
2. **Verify schema completeness:** grep for all 9 field names in the artifact_evaluation schema ✅
3. **Verify error fallback:** grep for `evaluation_error` schema with 3 fields ✅
4. **Verify rules section:** grep for "non-blocking", "secondary", "sequence suffix" keywords ✅
5. **Verify YAML well-formedness:** YAML blocks use proper fenced code block syntax (` ```yaml `) ✅

---

## Implementation Steps

1. Read design.md §Data Models & DTOs → §Artifact Evaluation Schema for the exact schema content
2. Read design.md §Storage Architecture → §Append-Only Semantics for collision avoidance rules
3. Create `.github/agents/evaluation-schema.md` with:
   - Title and version identifier
   - `artifact_evaluation` YAML schema with field descriptions in a fenced code block
   - `evaluation_error` fallback schema in a fenced code block
   - Rules section covering: one block per source artifact, list field minimums, score ranges, non-blocking behavior, secondary to primary output, collision avoidance (sequence suffix)
4. Verify all 9 schema fields are present with correct types and constraints
5. Verify YAML examples are syntactically well-formed

---

## Completion Checklist

- [x] File created at `.github/agents/evaluation-schema.md`
- [x] All 9 `artifact_evaluation` fields present with type constraints
- [x] `evaluation_error` fallback schema present
- [x] Rules section covers non-blocking, secondary output, collision avoidance
- [x] Version identifier included
- [x] YAML blocks are in fenced code blocks and well-formed
- [x] No existing files were modified
