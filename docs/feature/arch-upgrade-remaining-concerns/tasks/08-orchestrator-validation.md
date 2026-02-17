# Task 08: Add memory_access Validation Logic to Orchestrator

**Task Goal:** Add memory safety check validation at 5 parallel dispatch points in `orchestrator.agent.md` and update Step 5.2 between-wave memory update to capture documentation-writer outputs.

**depends_on:** 01

**agent:** implementer

---

## In-Scope

- Add memory safety check bullet at 5 parallel dispatch points in `orchestrator.agent.md`
- Update Step 5.2 between-wave memory update for documentation-writer outputs
- All edits are in `NewAgentsAndPrompts/orchestrator.agent.md` only

## Out-of-Scope

- Adding `memory_access` to orchestrator's own YAML frontmatter (it manages memory directly, not a sub-agent)
- Modifying any sub-agent files
- Modifying validation strictness (warnings only, no hard blocks — per design)

---

## Acceptance Criteria

1. The string `memory_access` appears at least 5 times in orchestrator validation contexts
2. Validation logic exists at Steps 1.1, 3b.1, 5.2, 6.2, and 7.2
3. Each validation point checks YAML `memory_access` field before parallel dispatch
4. Step 5.2 between-wave instruction includes documentation-writer Artifact Index and Recent Updates capture
5. Orchestrator's own YAML frontmatter does NOT contain `memory_access`

---

## Estimated Effort

Medium

---

## Test Requirements

- `grep -c "memory_access" NewAgentsAndPrompts/orchestrator.agent.md` → ≥5
- Verify textually that validation bullets exist at all 5 dispatch points
- Step 5.2 between-wave text mentions documentation-writer

---

## Implementation Steps

### Step 1: Add Validation Bullet at Step 1.1 (Research Focused Parallel Dispatch)

Insert before the dispatch instruction for the 4 focused researchers:

```markdown
- **Memory safety check:** Before dispatch, verify each agent's `memory_access` YAML field is `read-only`. Log warning if any agent is `read-write` or missing the field. Proceed regardless.
```

**Agents validated:** researcher ×4

### Step 2: Add Validation Bullet at Step 3b.1 (CT Cluster Parallel Dispatch)

Insert before the CT sub-agent dispatch instruction:

```markdown
- **Memory safety check:** Before dispatch, verify each agent's `memory_access` YAML field is `read-only`. Log warning if any agent is `read-write` or missing the field. Proceed regardless.
```

**Agents validated:** ct-security, ct-scalability, ct-maintainability, ct-strategy

### Step 3: Add Validation Bullet at Step 5.2 (Implementation Wave Parallel Dispatch)

Insert before the implementation wave dispatch instruction:

```markdown
- **Memory safety check:** Before dispatch, verify each agent's `memory_access` YAML field is `read-only`. Log warning if any agent is `read-write` or missing the field. Proceed regardless.
```

**Agents validated:** implementer ×N, documentation-writer ×N (all read-only)

### Step 4: Update Step 5.2 Between-Wave Memory Update

Find the existing between-wave instruction (approximately: "Between waves: Extract Lessons Learned from completed task outputs and append to `memory.md` (sequential — safe).").

**Replace with:**

```markdown
Between waves: Extract Lessons Learned from completed task outputs and append to `memory.md`. For documentation-writer outputs, also add Artifact Index entries (path and key sections) and a summary in Recent Updates. (Sequential — safe.)
```

### Step 5: Add Validation Bullet at Step 6.2 (V Cluster Parallel Dispatch)

Insert before the V sub-agent parallel dispatch (after v-build gate):

```markdown
- **Memory safety check:** Before dispatch, verify each agent's `memory_access` YAML field is `read-only`. Log warning if any agent is `read-write` or missing the field. Proceed regardless.
```

**Agents validated:** v-tests, v-tasks, v-feature

### Step 6: Add Validation Bullet at Step 7.2 (R Cluster Parallel Dispatch)

Insert before the R sub-agent dispatch:

```markdown
- **Memory safety check:** Before dispatch, verify each agent's `memory_access` YAML field is `read-only`. Log warning if any agent is `read-write` or missing the field. Proceed regardless.
```

**Agents validated:** r-quality, r-security, r-testing, r-knowledge

### Step 7: Verify

- Run: `grep -c "memory_access" NewAgentsAndPrompts/orchestrator.agent.md` → expect ≥5
- Confirm orchestrator YAML frontmatter does NOT contain `memory_access`
- Confirm Step 5.2 between-wave text includes documentation-writer

---

## Completion Checklist

- [x] Validation bullet added at Step 1.1
- [x] Validation bullet added at Step 3b.1
- [x] Validation bullet added at Step 5.2
- [x] Validation bullet added at Step 6.2
- [x] Validation bullet added at Step 7.2
- [x] Step 5.2 between-wave update expanded for documentation-writer outputs
- [x] Orchestrator YAML frontmatter does NOT contain `memory_access`
- [x] All grep checks pass
