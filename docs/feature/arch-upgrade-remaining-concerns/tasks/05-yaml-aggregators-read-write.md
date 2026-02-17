# Task 05: Add memory_access: read-write to Aggregator Agents

**Task Goal:** Add `memory_access: read-write` to the YAML frontmatter of all 3 aggregator agent files.

**depends_on:** none

**agent:** implementer

---

## In-Scope

- Add `memory_access: read-write` to YAML frontmatter of:
  1. `NewAgentsAndPrompts/ct-aggregator.agent.md`
  2. `NewAgentsAndPrompts/v-aggregator.agent.md`
  3. `NewAgentsAndPrompts/r-aggregator.agent.md`

## Out-of-Scope

- Modifying any file body content
- Modifying parallel sub-agents (handled by Tasks 02–04)

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

1. **ct-aggregator.agent.md** — The only CT cluster agent that writes to memory
2. **v-aggregator.agent.md** — The only V cluster agent that writes to memory
3. **r-aggregator.agent.md** — The only R cluster agent that writes to memory

---

## Completion Checklist

- [x] ct-aggregator.agent.md has `memory_access: read-write`
- [x] v-aggregator.agent.md has `memory_access: read-write`
- [x] r-aggregator.agent.md has `memory_access: read-write`
- [x] No other content modified in any file
