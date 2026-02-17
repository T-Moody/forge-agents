# Task 02: Add memory_access: read-only to CT Sub-Agents

**Task Goal:** Add `memory_access: read-only` to the YAML frontmatter of all 4 CT cluster sub-agent files.

**depends_on:** none

**agent:** implementer

---

## In-Scope

- Add `memory_access: read-only` to YAML frontmatter of:
  1. `NewAgentsAndPrompts/ct-security.agent.md`
  2. `NewAgentsAndPrompts/ct-scalability.agent.md`
  3. `NewAgentsAndPrompts/ct-maintainability.agent.md`
  4. `NewAgentsAndPrompts/ct-strategy.agent.md`

## Out-of-Scope

- Modifying any file body content
- Modifying `ct-aggregator.agent.md` (handled by Task 05)
- Modifying `critical-thinker.agent.md` (deprecated — do NOT touch)

---

## Acceptance Criteria

1. All 4 files contain `memory_access: read-only` in their YAML frontmatter
2. The `memory_access` field is the third field, after `name` and `description`
3. No other content in any file is modified
4. `critical-thinker.agent.md` is NOT modified

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

1. **ct-security.agent.md** — Open, find YAML frontmatter, add `memory_access: read-only` as third field
2. **ct-scalability.agent.md** — Same edit
3. **ct-maintainability.agent.md** — Same edit
4. **ct-strategy.agent.md** — Same edit

---

## Completion Checklist

- [x] ct-security.agent.md has `memory_access: read-only`
- [x] ct-scalability.agent.md has `memory_access: read-only`
- [x] ct-maintainability.agent.md has `memory_access: read-only`
- [x] ct-strategy.agent.md has `memory_access: read-only`
- [x] No other content modified in any file
- [x] critical-thinker.agent.md NOT touched

**Status:** Complete
**TDD skipped:** Configuration-only task (YAML metadata addition, no behavioral code)
**Note:** VS Code reports lint warnings for `memory_access` not being a supported `.agent.md` attribute — this is expected, as `memory_access` is a custom orchestrator-system metadata field, not a native VS Code attribute.
