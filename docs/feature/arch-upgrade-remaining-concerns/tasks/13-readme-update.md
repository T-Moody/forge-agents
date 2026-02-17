# Task 13: Update README.md

**Task Goal:** Update `README.md` to reflect the current architecture: ×4 researchers, full agent roster, and reference to `dispatch-patterns.md`.

**depends_on:** 12

**agent:** documentation-writer

---

## In-Scope

- Fix stale researcher count references (×3 → ×4)
- Update "Why Forge?" table concurrent agent count
- Update "Stages at a Glance" table
- Update workflow diagram (if present)
- Update Project Layout / Agent Roster to list all agent files
- Add reference to `NewAgentsAndPrompts/dispatch-patterns.md`

## Out-of-Scope

- Modifying any agent files
- Modifying files under `docs/feature/`
- Adding new sections unrelated to the architecture changes

---

## Acceptance Criteria

1. No stale references to "×3 researchers" or "3 concurrent" remain in README
2. All references to researcher count show ×4
3. Project Layout lists all cluster sub-agent files (ct-_, v-_, r-\* agents + aggregators)
4. `dispatch-patterns.md` is mentioned in the project layout or architecture section
5. No unrelated content is modified

---

## Estimated Effort

Low

---

## Test Requirements

- `grep "×3" README.md` → no researcher-related matches
- `grep "x3" README.md` → no researcher-related matches
- `grep "3 concurrent" README.md` → no matches
- `grep "×4" README.md` or `grep "4" README.md` → researcher references present
- `grep "dispatch-patterns" README.md` → match

---

## Implementation Steps

### Step 1: Read Current README

Read `README.md` fully to identify all sections that need updates.

### Step 2: Fix "Why Forge?" Table

Find "3 concurrent agents" (or similar) in the research phase row.

**Change to:** "4 concurrent agents"

### Step 3: Fix "Stages at a Glance" Table

Find "researcher ×3" or similar.

**Change to:** "researcher ×4"

### Step 4: Fix Workflow Diagram

If there is an ASCII or Mermaid diagram showing 3 researcher boxes, update to 4.

### Step 5: Update Project Layout / Agent Roster

Ensure all agent files in `NewAgentsAndPrompts/` are listed, organized by role:

- **Core pipeline agents:** researcher, spec, designer, planner, implementer, documentation-writer
- **CT cluster:** ct-security, ct-scalability, ct-maintainability, ct-strategy, ct-aggregator
- **V cluster:** v-build, v-tests, v-tasks, v-feature, v-aggregator
- **R cluster:** r-quality, r-security, r-testing, r-knowledge, r-aggregator
- **Reference docs:** dispatch-patterns.md
- **Prompt:** feature-workflow.prompt.md

### Step 6: Grep Sweep

Run grep to catch any remaining stale references:

- `grep -in "×3\|x3" README.md` — check for researcher-related matches
- `grep -in "3 concurrent" README.md` — should return no matches
- Fix any additional stale references found

### Step 7: Verify

Read through the updated sections to ensure accuracy and consistency.

---

## Completion Checklist

- [x] "Why Forge?" table updated (4 concurrent agents)
- [x] "Stages at a Glance" table updated (researcher ×4)
- [x] Workflow diagram updated (if applicable)
- [x] Project Layout lists all 22 agent files + dispatch-patterns.md
- [x] Grep sweep for stale ×3/x3/3-concurrent references passes
- [x] No unrelated content modified
