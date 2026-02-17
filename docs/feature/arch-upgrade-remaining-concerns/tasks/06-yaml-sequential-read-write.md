# Task 06: Add memory_access: read-write to Sequential Pipeline Agents

**Task Goal:** Add `memory_access: read-write` to the YAML frontmatter of the 3 sequential pipeline agents that write to memory.

**depends_on:** none

**agent:** implementer

---

## In-Scope

- Add `memory_access: read-write` to YAML frontmatter of:
  1. `NewAgentsAndPrompts/spec.agent.md`
  2. `NewAgentsAndPrompts/designer.agent.md`
  3. `NewAgentsAndPrompts/planner.agent.md`

## Out-of-Scope

- Modifying any file body content
- Modifying `implementer.agent.md` (handled by Task 07 — it is read-only)
- Modifying `documentation-writer.agent.md` (handled by Task 07 — it is read-only per design revision)
- Modifying `researcher.agent.md` (handled by Task 07)

---

## Acceptance Criteria

1. All 3 files contain `memory_access: read-write` in their YAML frontmatter
2. The `memory_access` field is the third field, after `name` and `description`
3. No other content in any file is modified

---

## Estimated Effort

Low

---

## Test Requirements

- For each of the 3 files: `grep "memory_access: read-write" <file>` returns exactly 1 match

---

## Implementation Steps

For each of the 3 files, modify the YAML frontmatter from:

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
memory_access: read-write
---
```

### Files to Edit

1. **spec.agent.md** — Sequential Step 2 agent; writes to memory at step 7
2. **designer.agent.md** — Sequential Step 3 agent; writes to memory at step 12
3. **planner.agent.md** — Sequential Step 4 agent; writes to memory at step 11

---

## Completion Checklist

- [x] spec.agent.md has `memory_access: read-write`
- [x] designer.agent.md has `memory_access: read-write`
- [x] planner.agent.md has `memory_access: read-write`
- [ ] No other content modified in any file
