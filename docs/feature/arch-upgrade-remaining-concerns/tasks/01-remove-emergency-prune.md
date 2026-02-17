# Task 01: Remove Emergency Prune from Orchestrator

**Task Goal:** Remove the 200-line emergency memory prune rule from `orchestrator.agent.md` — all 3 occurrences — and replace with explicit structural growth control language.

**depends_on:** none

**agent:** implementer

---

## In-Scope

- Remove emergency prune clause from Global Rule 6
- Remove emergency prune row from Memory Lifecycle Actions table
- Remove "emergency prune" from Anti-Drift Anchor
- Add explicit pruning semantics to Global Rule 6

## Out-of-Scope

- Modifying any file other than `NewAgentsAndPrompts/orchestrator.agent.md`
- Changing the `~200 lines per call` read_file guideline in any agent file (unrelated)
- Modifying historical docs under `docs/feature/forge-architecture-upgrade/`
- Adding any new pruning mechanism

---

## Acceptance Criteria

1. The string `emergency prune` (case-insensitive) does not appear anywhere in `orchestrator.agent.md`
2. The string `200 lines` does not appear in `orchestrator.agent.md` in any memory-pruning context
3. Memory Lifecycle Actions table has exactly 6 data rows (Initialize, Prune, Extract Lessons, Invalidate on revision, Clean invalidated, Validate)
4. Global Rule 6 still contains: `Initialize memory.md at Step 0`, `Prune memory at pipeline checkpoints`, `Invalidate memory entries on step failure/revision`
5. No other files are modified

---

## Estimated Effort

Low

---

## Test Requirements

- `grep -i "emergency prune" NewAgentsAndPrompts/orchestrator.agent.md` → 0 results
- `grep "200 lines" NewAgentsAndPrompts/orchestrator.agent.md` → no results in memory-pruning context
- Count Memory Lifecycle Actions table data rows → exactly 6

---

## Implementation Steps

### Step 1: Edit Global Rule 6 (~L34)

**Find this text:**

```
6. **Memory-First Protocol:** Initialize `memory.md` at Step 0. Prune memory at pipeline checkpoints (after Steps 1.2, 2, 4). Invalidate memory entries on step failure/revision. Emergency prune if `memory.md` exceeds 200 lines (keep only Lessons Learned + Artifact Index + current-phase entries). Memory failure is non-blocking — if `memory.md` cannot be created or becomes corrupted, log a warning and proceed. Agents fall back to direct artifact reads.
```

**Replace with:**

```
6. **Memory-First Protocol:** Initialize `memory.md` at Step 0. Prune memory at pipeline checkpoints (after Steps 1.2, 2, 4) — remove Recent Decisions and Recent Updates older than the current and previous phase; preserve Artifact Index and Lessons Learned always. Invalidate memory entries on step failure/revision. Memory failure is non-blocking — if `memory.md` cannot be created or becomes corrupted, log a warning and proceed. Agents fall back to direct artifact reads.
```

### Step 2: Delete Emergency Prune Row from Memory Lifecycle Actions Table (~L416)

**Find and remove this row:**

```
| Emergency prune        | When memory exceeds 200 lines              | Remove all except Lessons Learned + Artifact Index + current-phase entries                                                                   |
```

The table should retain 6 rows: Initialize, Prune, Extract Lessons, Invalidate on revision, Clean invalidated, Validate.

### Step 3: Edit Anti-Drift Anchor (~L485)

**Find:**

```
You manage the memory lifecycle (init, prune, invalidate, emergency prune).
```

**Replace with:**

```
You manage the memory lifecycle (init, prune, invalidate).
```

### Step 4: Verify

- Run: `grep -ic "emergency prune" NewAgentsAndPrompts/orchestrator.agent.md` → expect `0`
- Confirm no other files were modified

---

## Completion Checklist

- [x] Global Rule 6 updated — emergency prune clause removed, explicit pruning semantics added
- [x] Emergency prune row removed from Memory Lifecycle Actions table
- [x] Anti-Drift Anchor updated — "emergency prune" removed from lifecycle list
- [x] Table has exactly 6 data rows
- [x] No other files modified
- [x] All grep checks pass
