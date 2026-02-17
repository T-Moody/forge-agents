# Task 04: Add memory_access: read-only to R Sub-Agents

**Task Goal:** Add `memory_access: read-only` to the YAML frontmatter of all 4 R cluster sub-agent files.

**depends_on:** none

**agent:** implementer

---

## In-Scope

- Add `memory_access: read-only` to YAML frontmatter of:
  1. `NewAgentsAndPrompts/r-quality.agent.md`
  2. `NewAgentsAndPrompts/r-security.agent.md`
  3. `NewAgentsAndPrompts/r-testing.agent.md`
  4. `NewAgentsAndPrompts/r-knowledge.agent.md`

## Out-of-Scope

- Modifying any file body content (r-knowledge body changes are in Task 10)
- Modifying `r-aggregator.agent.md` (handled by Task 05)

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

1. **r-quality.agent.md** — Open, find YAML frontmatter, add `memory_access: read-only` as third field
2. **r-security.agent.md** — Same edit
3. **r-testing.agent.md** — Same edit
4. **r-knowledge.agent.md** — YAML frontmatter only; do NOT modify body content (that is Task 10)

---

## Completion Checklist

- [x] r-quality.agent.md has `memory_access: read-only`
- [x] r-security.agent.md has `memory_access: read-only`
- [x] r-testing.agent.md has `memory_access: read-only`
- [x] r-knowledge.agent.md has `memory_access: read-only` (body unchanged)
- [x] No other content modified in any file

## Status: COMPLETE

TDD skipped: configuration-only task (YAML frontmatter metadata addition, no behavioral code).

Note: VS Code reports `memory_access` as unsupported in `.agent.md` frontmatter — this is expected since it's a custom field for orchestrator use, not a native VS Code agent attribute.
