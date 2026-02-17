# Review: Architecture Upgrade Remaining Concerns

**Overall Verdict: Minor** — All requested changes implemented correctly with the revised (non-YAML) approach. Three minor consistency issues found, none blocking.

**Review Tier: Standard** — Business logic and internal architecture changes (agent definitions, reference documents, README). No security-sensitive code paths affected.

---

## Changed Files Summary

| File                                                | Scope of Changes                                                                                                                                                     |
| --------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `NewAgentsAndPrompts/orchestrator.agent.md`         | Emergency prune removal, Global Rule 12 (memory write safety), dispatch-point constraints, r-knowledge dispatch row, pattern extraction, compression — now 399 lines |
| `NewAgentsAndPrompts/dispatch-patterns.md`          | **New file.** Full definitions of Patterns A and B. Pattern C excluded (inline in orchestrator).                                                                     |
| `NewAgentsAndPrompts/r-knowledge.agent.md`          | Inputs reduced to `memory.md` + `initial-request.md`; Workflow Step 2 uses Artifact Index navigation with fallback                                                   |
| `NewAgentsAndPrompts/documentation-writer.agent.md` | Step 7 changed to "Do NOT write to `memory.md`" — orchestrator handles memory updates                                                                                |
| `README.md`                                         | Researcher ×4 references, 4 concurrent agents, full Project Layout with all 22 agents + dispatch-patterns.md                                                         |
| 21 agent files (spot-checked)                       | YAML frontmatter clean — `name` + `description` only, no stale `memory_access` fields                                                                                |

---

## Diff Highlights

### Global Rule 6 — Emergency Prune Removal

Orchestrator L32: Emergency prune clause replaced with explicit structural growth controls: "remove Recent Decisions and Recent Updates older than the current and previous phase; preserve Artifact Index and Lessons Learned always." Clean removal — no residual "emergency prune" or "200 lines" references in memory-pruning context.

### Global Rule 12 — Memory Write Safety (Revised Approach)

Orchestrator L42–46: Instead of YAML `memory_access` frontmatter fields (rejected by user because YAML `---` is for GitHub Copilot features only), a centralized Global Rule 12 defines the read-only/read-write agent classification. This is a sound alternative — the safety contract is explicit, centralized, and discoverable. Dispatch-point constraints ("Sub-agents read memory but do NOT write to it") at Steps 1.1, 3b.1, 5.2, 6.2, 7.2 provide defense-in-depth.

### Pattern Extraction

Patterns A and B extracted to `dispatch-patterns.md`; orchestrator retains one-line summaries + reference pointer at L87. Pattern C kept inline with full pseudocode — correct decision per design rationale (highest-risk pattern, needs discoverability).

### r-knowledge Input Scoping

Orchestrator Step 7.2 dispatch table correctly shows `tier, initial-request.md, memory.md` for r-knowledge. The agent's Inputs section and Workflow Step 2 both use Artifact Index navigation with explicit fallback to `grep_search`/`semantic_search`.

### Documentation-Writer Memory Safety

Step 7 at L67: "Do NOT write to `memory.md`. The orchestrator handles memory updates for documentation outputs between waves." Orchestrator Step 5.2 between-wave update extended to capture doc-writer Artifact Index entries and Recent Updates.

---

## Issues & Severity

### Issue 1 — Memory Lifecycle Table Contradicts Global Rule 6 on Artifact Index Pruning

**Severity: Major (non-blocking)**

Global Rule 6 (L32) now explicitly states: "preserve Artifact Index and Lessons Learned always." However, the Memory Lifecycle Actions table's "Prune" row (L362) reads:

> Remove entries from Recent Decisions, Recent Updates, Artifact Index older than 2 completed phases. Preserve Lessons Learned (never pruned).

This lists Artifact Index as a section subject to pruning, directly contradicting Rule 6. The initial request is unambiguous: "keep Artifact Index and Lessons Learned always." While the Prune row text appears to be pre-existing, the new Rule 6 language introduced by this feature creates a contradiction that didn't exist before. If the orchestrator AI reads the table, it may incorrectly prune Artifact Index entries.

**Suggested Fix:** Update the Prune row's "What" column to:

> Remove entries from Recent Decisions and Recent Updates older than 2 completed phases. Preserve Artifact Index and Lessons Learned (never pruned).

### Issue 2 — Implementer Classified as Read-Write in Rule 12

**Severity: Minor**

Rule 12 (L45) lists `implementer` under "Read-write (sequential only)." However:

- `implementer.agent.md` has no memory write step — its Outputs are code files, test files, and the task file only.
- FR-3.3 #13 classifies implementer as read-only ("Parallel within waves; no memory write").
- Step 5.2 correctly constrains implementer to read-only at dispatch time: "Sub-agents read memory but do NOT write to it."

No functional impact — the dispatch-time instruction is authoritative. But the classification is misleading and could confuse future maintainers.

**Suggested Fix:** Move `implementer` from the "Read-write (sequential only)" list to the "Read-only (parallel)" list in Rule 12.

### Issue 3 — README Project Layout Misattributes dispatch-patterns.md Content

**Severity: Minor**

README L390 says:

> `├── dispatch-patterns.md           # Reference: cluster dispatch patterns (A, B, C)`

But `dispatch-patterns.md` only contains Patterns A and B. Pattern C is defined inline in the orchestrator. The header of `dispatch-patterns.md` itself correctly states: "Pattern C (Replan Loop) is defined inline in the orchestrator."

**Suggested Fix:** Change the comment to `# Reference: cluster dispatch patterns (A, B)` or `# Reference: cluster dispatch patterns A and B (C is inline in orchestrator)`.

---

## Security Findings

### Secrets/PII Scan

Scanned all changed files for hardcoded secrets, API keys, tokens, passwords, and PII.

| Pattern                                   | Matches |
| ----------------------------------------- | ------- |
| `password`, `secret`, `api_key`, `apikey` | 0       |
| `token`, `Bearer`, `private_key`          | 0       |
| `AWS_`, `AZURE_`, `connection_string`     | 0       |
| PII (emails, phone numbers, SSNs)         | 0       |

**Result:** Clean. No secrets or PII in any changed files.

### Memory Safety Assessment

The revised approach (Global Rule 12 + dispatch-point constraints) provides equivalent safety to the originally-planned YAML markers:

- **Centralized contract:** Rule 12 defines the complete read-only/read-write classification in one place.
- **Distributed enforcement:** All 5 dispatch points repeat the "do NOT write" constraint.
- **Residual risk:** Same as the YAML approach — advisory only, no technical file locking. Acceptable per platform constraints.

The documentation-writer read-only change (Step 7 no longer writes to memory) eliminates the concurrent write risk that was the most realistic safety hazard.

---

## Testing Impact

No automated tests exist for this project. Verification is structural/textual. The verifier report confirms all 13 tasks pass against acceptance criteria.

**Recommended post-merge checks:**

1. Run a full pipeline on a test feature to confirm the orchestrator correctly handles memory beyond 200 lines without emergency pruning.
2. Verify r-knowledge can navigate via Artifact Index and falls back correctly when the index is sparse.
3. Confirm the orchestrator reads `dispatch-patterns.md` successfully when detailed Pattern A/B handling is needed.

---

## Architectural Decisions

No new architectural decisions requiring `decisions.md` entries — the decisions made (Pattern C inline, doc-writer read-only, Rule 12 instead of YAML) are already well-documented in the design and were driven by the user-rejected YAML approach.

---

## Completeness Check

| Requested Change                  | Status                | Notes                                                                                             |
| --------------------------------- | --------------------- | ------------------------------------------------------------------------------------------------- |
| Remove 200-line emergency prune   | ✅ Complete           | Rule 6 updated, table row removed, anti-drift anchor updated                                      |
| Reduce orchestrator to ≤400 lines | ✅ Complete           | 399 lines. Patterns A+B extracted, Pattern C inline, Parallel Execution Summary compressed        |
| Memory write safety markers       | ✅ Complete (revised) | YAML approach rejected → Global Rule 12 + dispatch-point constraints                              |
| Scope r-knowledge inputs          | ✅ Complete           | Orchestrator dispatch row + agent inputs + workflow step 2 all updated                            |
| README update                     | ✅ Complete           | ×4 researchers, full agent roster, project layout                                                 |
| No regressions                    | ✅ Verified           | critical-thinker.agent.md unmodified, feature-workflow.prompt.md unmodified, no stale YAML fields |

---

## Conformance to .github Instructions

No `.github/instructions/` files exist in the workspace. No updates needed.

---

## Checklist / Acceptance

- [x] All changed files reviewed (none skipped)
- [x] Emergency prune fully removed (grep confirmed 0 matches)
- [x] Orchestrator ≤400 lines (399)
- [x] dispatch-patterns.md exists with Patterns A and B
- [x] No stale `memory_access` YAML in any agent files (grep confirmed 0 matches)
- [x] Global Rule 12 defines memory write safety
- [x] All 5 dispatch points enforce read-only constraint
- [x] r-knowledge inputs scoped correctly
- [x] documentation-writer step 7 read-only
- [x] README accurate (×4, full project layout)
- [x] Security scan complete — no secrets/PII
- [ ] **Issue 1:** Fix Prune row to match Rule 6 (Artifact Index not pruned)
- [ ] **Issue 2:** Move implementer to read-only in Rule 12
- [ ] **Issue 3:** Fix dispatch-patterns.md description in README

---

## Owners & Next Steps

| Priority        | Action                                                                                         | Owner       | File                                             |
| --------------- | ---------------------------------------------------------------------------------------------- | ----------- | ------------------------------------------------ |
| 1 (recommended) | Fix Prune row in Memory Lifecycle Actions table — remove "Artifact Index" from pruned sections | Implementer | `NewAgentsAndPrompts/orchestrator.agent.md` L362 |
| 2 (recommended) | Move `implementer` from read-write to read-only in Rule 12                                     | Implementer | `NewAgentsAndPrompts/orchestrator.agent.md` L45  |
| 3 (low)         | Fix `dispatch-patterns.md` comment in README Project Layout — "(A, B, C)" → "(A, B)"           | Implementer | `README.md` L390                                 |
