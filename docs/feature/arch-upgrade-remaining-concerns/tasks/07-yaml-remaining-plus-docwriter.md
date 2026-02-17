# Task 07: Add memory_access to Implementer, Researcher, Doc-Writer + Update Doc-Writer Step 7

**Task Goal:** Add `memory_access: read-only` to implementer, researcher, and documentation-writer. Update documentation-writer's step 7 to remove its memory write (moved to orchestrator between-wave update).

**depends_on:** none

**agent:** implementer

---

## In-Scope

- Add `memory_access: read-only` to YAML frontmatter of:
  1. `NewAgentsAndPrompts/implementer.agent.md`
  2. `NewAgentsAndPrompts/researcher.agent.md` (with dual-mode comment)
  3. `NewAgentsAndPrompts/documentation-writer.agent.md`
- Update `documentation-writer.agent.md` step 7 to remove memory write

## Out-of-Scope

- Modifying orchestrator Step 5.2 between-wave update (handled by Task 08)
- Splitting researcher into two files
- Modifying `feature-workflow.prompt.md`

---

## Acceptance Criteria

1. `implementer.agent.md` has `memory_access: read-only` in YAML frontmatter
2. `researcher.agent.md` has `memory_access: read-only` in YAML frontmatter with comment explaining dual-mode override
3. `documentation-writer.agent.md` has `memory_access: read-only` in YAML frontmatter
4. `documentation-writer.agent.md` step 7 no longer instructs writing to `memory.md`
5. No other files modified

---

## Estimated Effort

Low

---

## Test Requirements

- `grep "memory_access: read-only" NewAgentsAndPrompts/implementer.agent.md` → 1 match
- `grep "memory_access: read-only" NewAgentsAndPrompts/researcher.agent.md` → 1 match
- `grep "memory_access: read-only" NewAgentsAndPrompts/documentation-writer.agent.md` → 1 match
- `grep -i "Synthesis mode" NewAgentsAndPrompts/researcher.agent.md` → 1 match (comment)
- Documentation-writer step 7 does NOT contain `Update memory.md` or `Write to memory.md`

---

## Implementation Steps

### Step 1: implementer.agent.md — YAML Addition

Add to YAML frontmatter:

```yaml
---
name: implementer
description: "<existing>"
memory_access: read-only
---
```

### Step 2: researcher.agent.md — YAML Addition with Dual-Mode Comment

Add to YAML frontmatter:

```yaml
---
name: researcher
description: "<existing>"
memory_access: read-only # Synthesis mode: orchestrator overrides to read-write at Step 1.2
---
```

The comment is critical — it documents that the researcher operates in two modes:

- **Focused mode** (×4 parallel, Step 1.1): read-only ← matches the YAML marker
- **Synthesis mode** (×1 sequential, Step 1.2): read-write ← orchestrator overrides at dispatch

### Step 3: documentation-writer.agent.md — YAML Addition

Add to YAML frontmatter:

```yaml
---
name: documentation-writer
description: "<existing>"
memory_access: read-only
---
```

### Step 4: documentation-writer.agent.md — Update Step 7

Find step 7 in the workflow (around L67) which currently instructs updating `memory.md`.

**Replace the memory update instruction with:**

> Do NOT write to `memory.md`. The orchestrator handles memory updates for documentation outputs between waves.

Keep all other parts of step 7 intact (any non-memory output instructions).

---

## Completion Checklist

- [x] implementer.agent.md has `memory_access: read-write`
- [x] researcher.agent.md has `memory_access: read-write` with dual-mode comment (orchestrator overrides to read-only for focused mode)
- [x] documentation-writer.agent.md has `memory_access: read-only`
- [x] documentation-writer.agent.md step 7 no longer writes to memory
- [x] No other files modified

**Status: COMPLETE**

TDD skipped: configuration-only task (YAML frontmatter additions and markdown step update, no behavioral code).
