# Task 04: Update Feature-Workflow Prompt

**Task Goal:** Update `.github/prompts/feature-workflow.prompt.md` to add references to evaluation artifacts, metrics, PostMortem step, and evaluation schema.

**depends_on:** none

**agent:** implementer

---

## In-Scope

- Modify `.github/prompts/feature-workflow.prompt.md`:
  - Add 3 new entries to the Key Artifacts table: `artifact-evaluations/`, `agent-metrics/`, `post-mortems/`
  - Add 2 new entries to the Rules section: PostMortem non-blocking rule, artifact evaluations rule

## Out-of-Scope

- Modifying any agent definition files
- Creating the evaluation schema or PostMortem agent
- Any structural changes to the prompt file beyond the specified additions

---

## Acceptance Criteria

1. Key Artifacts table contains entries for `artifact-evaluations/`, `agent-metrics/`, `post-mortems/` with accurate descriptions
2. Rules section contains a PostMortem rule: "After the R cluster completes (Step 7), the orchestrator dispatches a PostMortem agent (Step 8) that is non-blocking"
3. Rules section contains an artifact evaluations rule: "Agents that consume upstream pipeline artifacts produce structured evaluations in `artifact-evaluations/` following the schema in `.github/agents/evaluation-schema.md`"
4. Existing content in the file is unchanged — additions only
5. Additions match design.md §Feature Workflow Prompt Update text

---

## Estimated Effort

Low (~20 lines added to 1 file)

---

## Test Requirements (TDD Fallback — Configuration-Only)

1. **Verify Key Artifacts additions:** grep for `artifact-evaluations/`, `agent-metrics/`, `post-mortems/` in the file
2. **Verify Rules additions:** grep for "PostMortem" or "Post-mortem" and "evaluation-schema.md" in Rules section
3. **Verify no deletions:** existing Key Artifacts and Rules entries still present

---

## Implementation Steps

1. Read current `.github/prompts/feature-workflow.prompt.md` to understand structure
2. Read design.md §Feature Workflow Prompt Update for exact text to add
3. Locate the Key Artifacts table and add 3 new rows at the end
4. Locate the Rules section and add 2 new bullet points
5. Verify existing content is unchanged

---

## Completion Checklist

- [x] 3 new Key Artifacts entries added
- [x] 2 new Rules entries added
- [x] Existing content unchanged
- [x] Text matches design.md specification

TDD skipped: prompt file modification (configuration-only task)
