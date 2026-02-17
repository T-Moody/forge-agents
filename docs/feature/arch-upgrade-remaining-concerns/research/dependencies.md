# Research: Dependencies

- **Focus Area:** dependencies
- **Summary:** The 4 proposed changes share a single critical bottleneck file (`orchestrator.agent.md`, modified by all 4 changes) and have an explicit ordering dependency (Change 1 before Change 2). Change 3 touches all 22 agent files. Cross-references between agents are structural (aggregator references, memory write assertions) and will not break unless naming changes occur.

---

## Findings

### 1. Inter-Change Dependencies

#### 1.1 Change 1 → Change 2 (Explicit Dependency)

The initial request states Change 2 should "Simplify the Memory Lifecycle Actions table — especially now that emergency prune is removed (see item 1)." This is an **explicit prerequisite**: the emergency prune row must be removed (Change 1) before the Memory Lifecycle Actions table can be simplified (Change 2).

Specific coupling points in `orchestrator.agent.md`:

- **Global Rule 6** (line 34): Contains emergency prune clause ("Emergency prune if `memory.md` exceeds 200 lines…"). Change 1 modifies this. Change 2 depends on the simplified version.
- **Memory Lifecycle Actions table** (lines 408–417): The "Emergency prune" row (line 416) is removed by Change 1. Change 2's simplification of this table assumes that row is already gone.
- **Anti-drift anchor** (line 485): References "emergency prune" — Change 1 must update this, and Change 2's pattern extraction references the same anchor.

**Ordering constraint:** Change 1 must be applied before Change 2's Memory Lifecycle table simplification.

#### 1.2 Change 2 → Change 4 (Weak Dependency via Shared Edit Region)

Change 2 (Reduce Orchestrator Complexity) extracts Patterns A/B/C from `orchestrator.agent.md` into a separate `docs/patterns.md` file. Change 4 (Scope r-knowledge Inputs) modifies Step 7.2 in the orchestrator, which dispatches r-knowledge within Pattern A (R cluster).

If Pattern A is extracted before Change 4 is applied, then:

- The orchestrator's Step 7.2 dispatch table (lines 330–337) will reference the extracted patterns document instead of inlining Pattern A.
- Change 4's modifications to r-knowledge's input list in the dispatch table can still be applied independently because the dispatch table itself (agent names, inputs, outputs) stays in the orchestrator — only the pattern definitions move out.

**Ordering constraint:** Weak. Either order works, but applying Change 4 before Change 2 is slightly cleaner because Step 7.2's dispatch table is easier to edit while the full context is still inline.

#### 1.3 Change 3 → Change 2 (Opposing Line-Count Pressure)

Change 3 (Add Memory Write Safety Markers) **adds** validation logic to the orchestrator's dispatch steps ("if dispatching in parallel, all agents in the wave MUST have `memory_access: read-only`"). Change 2 (Reduce Orchestrator Complexity) **removes** lines from the orchestrator by extracting patterns. These create opposing pressure on the orchestrator's line count.

**Ordering constraint:** Change 2's ≤400-line target should be assessed **after** Change 3's additions. If Change 3 is applied first, the line count will increase before Change 2 reduces it. If Change 2 is applied first, the ≤400-line target may be met temporarily but then exceeded when Change 3 adds validation logic. The safer approach: apply Change 3 first, then Change 2 last (as the initial request's priority order already suggests).

#### 1.4 Change 1 ↔ Change 3 (No Dependency)

Removing memory line limits (Change 1) and adding `memory_access` frontmatter markers (Change 3) are fully independent. They touch different parts of the orchestrator (Global Rule 6/Memory Lifecycle table vs. dispatch validation logic) and don't share semantic concerns.

#### 1.5 Change 1 ↔ Change 4 (No Dependency)

Removing memory line limits (Change 1) and scoping r-knowledge inputs (Change 4) are fully independent. Change 1 modifies memory rules; Change 4 modifies r-knowledge's input list.

#### 1.6 Change 3 ↔ Change 4 (No Dependency, But Shared File)

Both changes touch `r-knowledge.agent.md` — Change 3 adds a `memory_access: read-write` frontmatter field, Change 4 modifies the Inputs section and Workflow Step 2. These are in non-overlapping regions (frontmatter vs. body) with no conflict.

#### Dependency Graph Summary

```
Change 1 (Remove Memory Limits)
    │
    ▼  [prerequisite for table simplification]
Change 2 (Reduce Orchestrator Complexity)

Change 3 (Memory Write Safety Markers) — independent of 1
Change 4 (Scope r-knowledge Inputs)     — independent of 1
Change 3 ←→ Change 4: no dependency, both touch r-knowledge.agent.md in non-overlapping regions
Change 3 → Change 2: apply before Change 2 to get accurate line count for ≤400 target
Change 4 → Change 2: apply before Change 2 for cleaner edits to Step 7.2
```

**Recommended application order:** 1 → 3 → 4 → 2 (matches initial request priority order: 1, 2→moved last, 3, 4, then 2 as final).

---

### 2. File-Level Dependencies (Multi-Change Collision Map)

#### 2.1 `orchestrator.agent.md` — ALL 4 CHANGES (Critical Bottleneck)

This is the highest-risk file. All 4 changes modify it:

| Change   | What's Modified                                                                                                                           | Lines Affected                             |
| -------- | ----------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------ |
| Change 1 | Global Rule 6 (line 34), Memory Lifecycle Actions table row (line 416), Anti-drift anchor (line 485)                                      | 3 distinct regions                         |
| Change 2 | Cluster Dispatch Patterns section (lines 84–118), Memory Lifecycle Actions table (lines 408–417), all Workflow Steps referencing patterns | ~60 lines extracted, net reduction         |
| Change 3 | New validation logic added to dispatch steps (1.1, 3b.1, 5.2, 6.2, 7.2)                                                                   | ~5–10 lines added across multiple sections |
| Change 4 | Step 7.2 R sub-agent dispatch table (lines 330–337), r-knowledge input row                                                                | 1–2 lines changed                          |

**Merge conflict risk:** HIGH. Changes 1, 2, 3, and 4 all touch this file. However, the edit regions are mostly non-overlapping:

- Change 1 edits Global Rule 6 + Memory Lifecycle table + anti-drift anchor
- Change 2 edits Pattern definitions section + Memory Lifecycle table + workflow step headers
- Change 3 edits dispatch step bodies (1.1, 3b.1, 5.2, 6.2, 7.2)
- Change 4 edits Step 7.2 dispatch table row

**Conflict zone:** Memory Lifecycle Actions table — modified by both Change 1 (remove row) and Change 2 (simplify table). If applied sequentially (1 first), no conflict.

#### 2.2 `r-knowledge.agent.md` — Changes 3 and 4

| Change   | What's Modified                                                                                                                      |
| -------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| Change 3 | YAML frontmatter: add `memory_access: read-write`                                                                                    |
| Change 4 | Inputs section (lines 17–28): reduce input list; Workflow Step 2 (line 78): change from full-file reads to Artifact Index navigation |

Non-overlapping edit regions. No conflict.

#### 2.3 All 22 Agent Files — Change 3 Only

Change 3 adds a `memory_access` frontmatter field to every agent file. These are the complete lists:

**Parallel sub-agents (`memory_access: read-only`):**

- ct-security.agent.md, ct-scalability.agent.md, ct-maintainability.agent.md, ct-strategy.agent.md
- v-tests.agent.md, v-tasks.agent.md, v-feature.agent.md
- r-quality.agent.md, r-security.agent.md, r-testing.agent.md, r-knowledge.agent.md
- researcher.agent.md (focused instances)

**Sequential/aggregator agents (`memory_access: read-write`):**

- ct-aggregator.agent.md, v-aggregator.agent.md, r-aggregator.agent.md, v-build.agent.md
- researcher.agent.md (synthesis instance — same file, dual mode)
- spec.agent.md, designer.agent.md, planner.agent.md, implementer.agent.md
- documentation-writer.agent.md, orchestrator.agent.md

**Special case — `researcher.agent.md`:** This file operates in both focused (parallel, read-only) and synthesis (sequential, read-write) modes. The frontmatter cannot express both. Either: (a) use `memory_access: read-only` and note that synthesis mode has write permission by orchestrator override, or (b) add a mode-conditional field like `memory_access_focused: read-only` + `memory_access_synthesis: read-write`.

**Special case — `v-build.agent.md`:** V-Build is dispatched sequentially (gate agent in Pattern B), but its workflow says "No memory write step" (line 115). It reads memory but does NOT write to it. This makes it `memory_access: read-only` despite being sequential.

#### 2.4 `README.md` — Updated After All Changes

The initial request says "Update README.md when finished." This file has no merge conflict risk because it's only updated once at the end.

#### 2.5 New File: `docs/patterns.md` — Change 2 Only

Change 2 creates this new file. No collision risk.

---

### 3. Agent Cross-References

#### 3.1 Aggregator References (Structural, Stable)

Every parallel sub-agent references its aggregator by behavior (not by filename invocation):

| Sub-Agent Group                                              | References                | Example Text                                                                                       |
| ------------------------------------------------------------ | ------------------------- | -------------------------------------------------------------------------------------------------- |
| V sub-agents (v-tests, v-tasks, v-feature, v-build)          | "V Aggregator"            | "The V Aggregator will consolidate relevant findings into memory after all V sub-agents complete." |
| R sub-agents (r-quality, r-security, r-testing, r-knowledge) | "R Aggregator"            | "The R Aggregator will consolidate relevant findings into memory after all R sub-agents complete." |
| CT sub-agents                                                | "CT Aggregator" (implied) | CT sub-agents reference output paths `ct-review/ct-*.md`                                           |

**Impact of proposed changes:** None of the 4 changes rename agents or change aggregator relationships. These cross-references remain stable.

#### 3.2 Memory Write/No-Write Assertions (Affected by Change 3)

The following agents explicitly assert their memory behavior in prose:

**"Do NOT write to memory.md" assertions found in:**

- v-tasks.agent.md (lines 17, 51, 59, 159)
- v-feature.agent.md (lines 17, 50, 58, 173)
- v-tests.agent.md (line 146)
- v-build.agent.md (line 117)
- r-quality.agent.md (lines 12, 114)
- r-security.agent.md (lines 14, 152, 228)
- r-testing.agent.md (line 151)
- r-knowledge.agent.md (line 255)
- ct-security.agent.md (line 138)
- ct-scalability.agent.md (line 139)
- ct-strategy.agent.md (line 149)

**"Writes to memory.md" assertions found in:**

- v-aggregator.agent.md (lines 10, 138, 253)
- r-aggregator.agent.md (lines 10, 157, 266)
- ct-aggregator.agent.md (implied via aggregator pattern)

**Relevance to Change 3:** The `memory_access` frontmatter marker duplicates the prose assertions already present. The prose assertions should remain for anti-drift anchor reinforcement; the frontmatter adds a machine-checkable contract. No contradiction — the frontmatter formalizes what the prose already states.

#### 3.3 Orchestrator as Dispatch Authority

The orchestrator references every agent by name in its dispatch tables and workflow steps. Sub-agents reference "the orchestrator" for:

- Retry budget explanation ("The orchestrator also retries entire agent invocations once (Global Rule 4)") — found in all sub-agents' Operating Rules.
- Review tier designation ("The orchestrator may provide a tier designation at dispatch") — found in r-quality.agent.md (line 50), r-security.agent.md (line 51).
- Focus area assignment ("provided by orchestrator in the prompt") — found in researcher.agent.md (line 24).

**Impact of proposed changes:** None of these reference patterns are affected by the 4 changes. The orchestrator dispatch table modifications (Changes 3 and 4) don't change how sub-agents reference the orchestrator.

#### 3.4 R-Knowledge Input Cross-References

R-knowledge's inputs (lines 17–28) list 6 artifact files plus codebase access. The orchestrator's Step 7.2 dispatch table (line 336) lists: `tier, initial-request.md, feature.md, design.md, plan.md, verifier.md`. There is a slight mismatch:

- **r-knowledge.agent.md** lists `memory.md` first, then `initial-request.md`, `feature.md`, `design.md`, `plan.md`, `verifier.md`, `decisions.md`, `.github/instructions/`, git diff, entire codebase.
- **orchestrator.agent.md** Step 7.2 lists: `tier, initial-request.md, feature.md, design.md, plan.md, verifier.md` (plus `memory.md` implicitly via "additional to memory.md" column header).

Change 4 would narrow both to `memory.md` + `initial-request.md`, with other files accessed via Artifact Index navigation. Both the orchestrator dispatch table and the r-knowledge Inputs section must be updated **consistently**.

---

### 4. README Dependencies

#### 4.1 Sections Requiring Update

| README Section                                             | Current Content                                                            | Issue                                                                            | Which Change                          |
| ---------------------------------------------------------- | -------------------------------------------------------------------------- | -------------------------------------------------------------------------------- | ------------------------------------- |
| "Why Forge?" table, row "Sequential bottlenecks" (line 17) | "Parallel research (3 concurrent agents)"                                  | Should say "4 concurrent agents" (already stale from forge-architecture-upgrade) | Pre-existing, fixable with any change |
| "Stages at a Glance" table, row 1 (line 113)               | "researcher ×3 + synthesis \| 3 concurrent"                                | Should say "×4 + synthesis \| 4 concurrent"                                      | Pre-existing                          |
| Workflow diagram (lines 67–98)                             | Shows 3 researcher boxes                                                   | Should show 4 (architecture, impact, dependencies, patterns)                     | Pre-existing                          |
| "What's New in v2" table                                   | No mention of memory system, dispatch patterns, or `memory_access` markers | New rows needed for Changes 1–4                                                  | All changes                           |
| "Key Design Decisions" section (lines 200+)                | No mention of memory system                                                | Could add a "Memory-First Protocol" decision                                     | Changes 1, 3                          |
| "Completion Contracts" section                             | Already accurate                                                           | No change needed                                                                 | N/A                                   |
| "Project Layout" section (lines 277–291)                   | Lists 10 agent files; does not list sub-agent files                        | May need updating to mention cluster sub-agents exist in `NewAgentsAndPrompts/`  | Pre-existing                          |

#### 4.2 README Does NOT Reference

The following are **not** mentioned in README, so no update needed for removal:

- 200-line memory limit / emergency prune rule
- Dispatch Pattern A/B/C by name
- `memory_access` frontmatter field
- r-knowledge's specific input list

#### 4.3 `feature-workflow.prompt.md` References

The feature-workflow prompt (lines 1–52) references:

- "Memory system: Initialize and maintain `memory.md` across all pipeline steps. Prune at checkpoints to keep context manageable." — No mention of emergency prune or 200-line limit. No update needed for Change 1.
- "Research runs 4 parallel sub-agents: architecture, dependencies, impact, and patterns." — Already correct.
- "Maximum 4 concurrent subagent invocations per wave." — Already correct.

**No changes required to `feature-workflow.prompt.md` for any of the 4 proposed changes.** It's sufficiently abstract.

---

### 5. Existing Documentation Dependencies

#### 5.1 `docs/feature/forge-architecture-upgrade/`

These are **historical** documents recording what was designed and implemented. They describe the system **as it was built**, including the 200-line emergency prune rule:

- [verifier.md](../forge-architecture-upgrade/verifier.md#L126): "Memory lifecycle: Step 0 initializes memory.md, pruning at checkpoints, invalidation on revision, emergency pruning >200 lines"
- [verifier.md](../forge-architecture-upgrade/verifier.md#L180): "MEM-AC-8 | Met | Emergency pruning at >200 lines"
- [tasks/task-14-orchestrator-rewrite.md](../forge-architecture-upgrade/tasks/task-14-orchestrator-rewrite.md#L138): "Emergency pruning when >200 lines"
- [review.md](../forge-architecture-upgrade/review.md#L105): References orchestrator at 473 lines exceeding 450-line task limit

**Recommendation:** These are historical artifacts documenting a completed feature. They should NOT be modified. The new feature's artifacts will document the changes.

#### 5.2 `docs/feature/agent-improvements/`

Historical artifacts from the earlier improvement feature. References "~200 lines per read_file call" (context-reading guidance) — this is a **different** 200-line reference (tool usage guideline, not memory limit). Not affected by Change 1.

#### 5.3 `docs/optimization-from-gem-team.md`

References cross-agent memory (Section 7) as a P2 improvement from Gem Team. Does not mention emergency pruning or 200-line limits. Not affected by any of the 4 changes.

#### 5.4 `docs/comparison-forge-vs-gem-team.md`

Compares Forge vs Gem Team architectures. References "Memory system: None — agents communicate through artifact files" in the Forge column (this was before the architecture upgrade). This is historically accurate for the time of writing. Not affected by the 4 changes.

---

## File References

| File/Folder                                                                           | Relevance                                                                |
| ------------------------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| [orchestrator.agent.md](../../../NewAgentsAndPrompts/orchestrator.agent.md)           | Modified by ALL 4 changes. Critical bottleneck. 489 lines currently.     |
| [r-knowledge.agent.md](../../../NewAgentsAndPrompts/r-knowledge.agent.md)             | Modified by Changes 3 (frontmatter) and 4 (inputs + workflow).           |
| [feature-workflow.prompt.md](../../../NewAgentsAndPrompts/feature-workflow.prompt.md) | Checked — no changes needed.                                             |
| [README.md](../../../README.md)                                                       | Updated after all changes. Has stale researcher count (3 vs 4).          |
| All 22 agent files in NewAgentsAndPrompts/                                            | Modified by Change 3 (frontmatter `memory_access` field).                |
| docs/feature/forge-architecture-upgrade/                                              | Historical artifacts. Reference emergency prune. Should not be modified. |
| docs/feature/agent-improvements/                                                      | Historical artifacts. Not affected.                                      |
| docs/optimization-from-gem-team.md                                                    | Not affected.                                                            |
| docs/patterns.md (new)                                                                | Created by Change 2. Does not yet exist.                                 |

---

## Assumptions & Limitations

1. **Assumed `memory_access` is a YAML frontmatter field.** The current frontmatter structure for all agents uses only `name` and `description` fields in a `---` delimited YAML block. Adding `memory_access` is a new field type — verified that this is the only frontmatter extension proposed.

2. **Assumed `researcher.agent.md` dual-mode issue must be resolved.** The researcher operates in both parallel (read-only) and sequential (read-write) modes from a single file. The `memory_access` field cannot capture this duality directly. This is an open design question for Change 3.

3. **Assumed v-build is read-only for memory despite being sequential.** V-Build's workflow explicitly says "No memory write step" even though it runs sequentially (as a gate agent). The `memory_access` field should be `read-only` for v-build, even though it's dispatched alone (not in parallel).

4. **Limited to the files in `NewAgentsAndPrompts/` and `docs/`.** Did not examine any `.github/` or source code files (none exist in this repository).

---

## Open Questions

1. **How should `researcher.agent.md` handle dual `memory_access` modes?** The file serves both parallel (focused, read-only) and sequential (synthesis, read-write) dispatch modes. Options: (a) default to `read-only` with orchestrator override for synthesis, (b) add mode-conditional fields, (c) split into two agent files.

2. **Should the orchestrator's `memory_access` validation be a hard error or a warning?** The initial request says "Log a warning if violated." If it's a warning only, the marker is advisory rather than enforced — reduces implementation complexity but also reduces safety value.

3. **Where exactly does the new validation logic for `memory_access` go in the orchestrator?** The initial request says "Update orchestrator to validate the `memory_access` field before dispatch." This could be: (a) a new Global Rule, (b) added to each dispatch step (1.1, 3b.1, 5.2, 6.2, 7.2), or (c) added to the Pattern A/B definitions. If patterns are extracted (Change 2), the validation might belong in the patterns document.

4. **Does Change 2's pattern extraction create a file-reading cost?** If patterns are in `docs/patterns.md`, the orchestrator must read that file. This adds a file read to the orchestrator's startup, but the orchestrator already reads multiple files.

5. **Should the README stale data (3 researchers → 4) be fixed as part of this feature or flagged separately?** It's a pre-existing issue from the forge-architecture-upgrade feature.

---

## Research Metadata

- **confidence_level:** high — All relevant files have been examined. The dependency graph is fully mapped. Cross-references were systematically searched.
- **coverage_estimate:** Comprehensive coverage of all 22 agent files in `NewAgentsAndPrompts/`, the README, the feature-workflow prompt, and all documentation under `docs/`. Every file in the workspace was checked for relevant references.
- **gaps:** None identified. The workspace contains only markdown agent definitions and documentation files — no source code, tests, or configuration files that could harbor hidden dependencies.
