# Task 03: Add memory_access: read-only to V Sub-Agents

**Task Goal:** Add `memory_access: read-only` to the YAML frontmatter of all 4 V cluster sub-agent files.

**depends_on:** none

**agent:** implementer

---

## In-Scope

- Add `memory_access: read-only` to YAML frontmatter of:
  1. `NewAgentsAndPrompts/v-build.agent.md`
  2. `NewAgentsAndPrompts/v-tests.agent.md`
  3. `NewAgentsAndPrompts/v-tasks.agent.md`
  4. `NewAgentsAndPrompts/v-feature.agent.md`

## Out-of-Scope

- Modifying any file body content
- Modifying `v-aggregator.agent.md` (handled by Task 05)

---

## Acceptance Criteria

1. All 4 files contain `memory_access: read-only` in their YAML frontmatter
2. The `memory_access` field is the third field, after `name` and `description`
3. No other content in any file is modified

---

## Estimated Effort

Low

---

## Test Requirements

- For each of the 4 files: `grep "memory_access: read-only" <file>` returns exactly 1 match
- YAML frontmatter structure is valid (3 fields: name, description, memory_access)

---

## Implementation Steps

For each of the 4 files, modify the YAML frontmatter from:

```yaml
---
name: <agent-name>
description: "<description>"
---
```

To:

```yaml
---
name: <agent-name>
description: "<description>"
memory_access: read-only
---
```

### Files to Edit

1. **v-build.agent.md** — Open, find YAML frontmatter, add `memory_access: read-only` as third field
2. **v-tests.agent.md** — Same edit
3. **v-tasks.agent.md** — Same edit
4. **v-feature.agent.md** — Same edit

**Note:** v-build is dispatched sequentially as a gate agent, but its workflow explicitly has "No Memory Write" — it is correctly classified as read-only.

---

## Completion Checklist

- [x] v-build.agent.md has `memory_access: read-only`
- [x] v-tests.agent.md has `memory_access: read-only`
- [x] v-tasks.agent.md has `memory_access: read-only`
- [x] v-feature.agent.md has `memory_access: read-only`
- [x] No other content modified in any file

## Status: COMPLETE
