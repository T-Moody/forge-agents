# Review: Agent-Isolated Memory System

**Overall Verdict:** Minor — 3 minor issues, 0 blockers, 0 major issues.

**Review Tier:** Standard — Business logic / architectural refactoring of agent prompt definitions. No runtime code, auth, payments, or networking. Security scan performed (standard secrets/PII scan).

---

## Changed Files Summary

| File                            | Change Scope              | Notes                                                                                                                                                                    |
| ------------------------------- | ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `ct-aggregator.agent.md`        | Deprecated                | Contains only deprecation notice. No active workflow. ✓                                                                                                                  |
| `v-aggregator.agent.md`         | Deprecated                | Contains only deprecation notice. No active workflow. ✓                                                                                                                  |
| `r-aggregator.agent.md`         | Deprecated                | Contains only deprecation notice. No active workflow. ✓                                                                                                                  |
| `orchestrator.agent.md`         | Major rewrite (483 lines) | Absorbs all aggregator decision logic, memory merge, removes Step 1.2, updates routing table, documentation structure, expectations table, parallel execution summary. ✓ |
| `researcher.agent.md`           | Major change              | Synthesis mode completely removed. Only focused mode remains. Isolated memory added. ✓                                                                                   |
| `spec.agent.md`                 | Moderate                  | Inputs updated to individual research files. Isolated memory added. Memory-first reading workflow step present. ✓                                                        |
| `designer.agent.md`             | Moderate                  | Inputs updated. CT review files for revision mode. Isolated memory added. ✓                                                                                              |
| `planner.agent.md`              | Moderate                  | Replan mode updated to use individual V artifacts. 6-step cross-referencing workflow (steps 0–5). Isolated memory added. ✓                                               |
| `r-security.agent.md`           | Moderate                  | Pipeline Blocker Override references orchestrator. Blocker/Major/Minor vocabulary with explicit "Do not use Critical" constraint. Isolated memory added. ✓               |
| `dispatch-patterns.md`          | Moderate                  | Patterns A, B, C updated — no aggregator steps. Memory-First Pattern section added. ✓                                                                                    |
| `feature-workflow.prompt.md`    | Moderate                  | No aggregator references. Isolated memory model described. Key Artifacts table updated. ✓                                                                                |
| `ct-security.agent.md`          | Minor                     | Isolated memory in Outputs + workflow step + Anti-Drift ✓                                                                                                                |
| `ct-scalability.agent.md`       | Minor                     | Isolated memory in Outputs + workflow step + Anti-Drift ✓                                                                                                                |
| `ct-maintainability.agent.md`   | Minor                     | Isolated memory in Outputs + workflow step + Anti-Drift ✓                                                                                                                |
| `ct-strategy.agent.md`          | Minor                     | Isolated memory in Outputs + workflow step + Anti-Drift ✓                                                                                                                |
| `v-build.agent.md`              | Minor                     | Isolated memory in Outputs + workflow step + Anti-Drift ✓                                                                                                                |
| `v-tests.agent.md`              | Minor                     | Isolated memory in Outputs + workflow step + Anti-Drift ✓                                                                                                                |
| `v-tasks.agent.md`              | Minor                     | Isolated memory in Outputs + workflow step + Anti-Drift ✓                                                                                                                |
| `v-feature.agent.md`            | Minor                     | Isolated memory in Outputs + workflow step + Anti-Drift ✓                                                                                                                |
| `r-quality.agent.md`            | Minor                     | Isolated memory in Outputs + workflow step + Anti-Drift ✓ (role description issue — see Issue 1)                                                                         |
| `r-testing.agent.md`            | Minor                     | Isolated memory in Outputs + workflow step + Anti-Drift ✓                                                                                                                |
| `r-knowledge.agent.md`          | Minor                     | Isolated memory in Outputs + workflow step + Anti-Drift ✓                                                                                                                |
| `implementer.agent.md`          | Minor                     | Isolated memory in Outputs + workflow step + Anti-Drift ✓                                                                                                                |
| `documentation-writer.agent.md` | Minor                     | Isolated memory in Outputs + workflow step + Anti-Drift ✓ (Inputs issue — see Issue 3)                                                                                   |
| `critical-thinker.agent.md`     | None                      | Already deprecated. No changes required. ✓                                                                                                                               |

---

## Issues & Severity

### Issue 1 — Minor: r-quality.agent.md role description uses old memory phrasing

**File:** `r-quality.agent.md`, line 12
**Finding:** The role description says:

> You do NOT write to `memory.md`.

This is the old-style unqualified prohibition that doesn't mention isolated memory. The Anti-Drift Anchor (line ~200) is correctly updated to reference isolated memory:

> You write only to your isolated memory file (`memory/r-quality.mem.md`), never to shared `memory.md`.

The role description at line 12 is the first thing the LLM reads after the identity statement. Having the old phrasing there without mentioning isolated memory could confuse the model about whether it writes ANY memory file at all.

**Suggested Fix:** Update line 12 from:

```
You NEVER modify source code, test files, or project files. You write review findings only. You do NOT write to `memory.md`.
```

to:

```
You NEVER modify source code, test files, or project files. You write review findings only. You write only to your isolated memory file (`memory/r-quality.mem.md`), never to shared `memory.md`.
```

**AC Impact:** AC-16 (Anti-Drift Anchors Updated) — the Anti-Drift Anchor itself passes, but the role description has a stale variant of the same prohibition.

---

### Issue 2 — Minor: CT cluster inconsistency in Inputs and Operating Rule 6

**Files:** `ct-security.agent.md`, `ct-scalability.agent.md`, `ct-maintainability.agent.md`, `ct-strategy.agent.md`

**Finding:** The 4 CT agents have inconsistent approaches to memory reading:

| Agent              | `memory.md` in Inputs? | Operating Rule 6                                                                                |
| ------------------ | ---------------------- | ----------------------------------------------------------------------------------------------- |
| ct-strategy        | Yes                    | Standard: "Read `memory.md` FIRST"                                                              |
| ct-security        | No                     | Customized: "Read upstream memory files FIRST (`memory/designer.mem.md`, `memory/spec.mem.md`)" |
| ct-scalability     | No                     | Customized: "Read upstream memory files FIRST (`memory/designer.mem.md`, `memory/spec.mem.md`)" |
| ct-maintainability | No                     | Customized: "Read upstream memory files FIRST (`memory/designer.mem.md`, `memory/spec.mem.md`)" |

The orchestrator at Step 3b.1 (line 268) states: "Each receives: `initial-request.md`, `design.md`, `feature.md`, `memory.md`." This matches ct-strategy but contradicts the other 3 CT agents, which don't list `memory.md` in their Inputs.

**Context:** This is a design choice issue, not a functional failure. Both approaches work:

- **Standard approach (ct-strategy):** Read shared `memory.md` for the full Artifact Index, orientation context from all prior steps.
- **Upstream-memory approach (ct-security, ct-scalability, ct-maintainability):** Read only the specific upstream memory files that matter (designer + spec). More targeted but loses broader pipeline context.

**Suggested Fix (Option A — align all CT agents to upstream-memory approach):**

- Remove `memory.md` from ct-strategy Inputs
- Update ct-strategy Operating Rule 6 to match the other 3 CT agents
- Update orchestrator line 268 to say "Each receives: `initial-request.md`, `design.md`, `feature.md`, `memory/designer.mem.md`, `memory/spec.mem.md`"

**Suggested Fix (Option B — align all CT agents to standard approach):**

- Add `memory.md` to ct-security, ct-scalability, ct-maintainability Inputs
- Update their Operating Rule 6 to the standard version

**AC Impact:** AC-19 (Convention Consistency) — technically passes since AC-19 requires Operating Rules 1–4 to be identical (Rule 6 is allowed to vary). However, within the CT cluster, sibling agents should be consistent with each other.

---

### Issue 3 — Minor: documentation-writer.agent.md missing memory.md in Inputs

**File:** `documentation-writer.agent.md`
**Finding:** Same pattern as Issue 2. The documentation-writer does not list `memory.md` in its Inputs section. It uses the upstream-memory variant of Operating Rule 6:

> Read upstream memory files FIRST (`memory/planner.mem.md`, `memory/designer.mem.md`, `memory/spec.mem.md`)

Meanwhile, `implementer.agent.md` (its sibling in the implementation wave) uses a hybrid approach:

> Read `memory.md` (maintained by orchestrator) FIRST, then read upstream memory files

And the implementer lists `memory.md` in its Inputs.

**Suggested Fix:** Align with implementer's approach — add `memory.md` to the documentation-writer Inputs section and update Operating Rule 6 to reference both `memory.md` and upstream memories. Or align both to the upstream-only approach.

**AC Impact:** Non-blocking. Both approaches are functional.

---

## Diff Highlights

### Orchestrator — Key Patterns Verified

1. **Global Rule 12 (Memory Write Safety):** Correctly states orchestrator is sole writer to `memory.md`. No aggregators listed. Sub-agents write only to isolated memory files. ✓
2. **Documentation Structure table:** `memory/` directory added. No `analysis.md`, `design_critical_review.md`, `verifier.md`, `review.md` entries. ✓
3. **Step 1.2 (Synthesize Research):** Completely removed. After research agents complete, orchestrator reads memories and merges. Proceeds directly to Step 2. ✓
4. **CT Cluster Decision Flow:** Reads CT memories, checks severity values, applies Critical/High → NEEDS_REVISION, all ≤Medium → DONE, <2 outputs → ERROR. Includes self-verification logging. ✓
5. **V Cluster Decision Flow:** Full decision table present (9 rows matching FR-4.2). Pattern C replan loop references `verification/v-tasks.md` (not `verifier.md`). Max 3 iterations preserved. ✓
6. **R Cluster Decision Flow:** R-Security pipeline override present (Blocker/Critical → ERROR). R-Security mandatory check. R-Knowledge explicitly non-blocking. Severity vocabulary mapping (Critical → Blocker for safety). ✓
7. **NEEDS_REVISION Routing Table:** Updated — "Orchestrator (CT cluster evaluation)", "Orchestrator (V cluster evaluation)", "Orchestrator (R cluster evaluation)". No aggregator references. ✓
8. **Memory Lifecycle Actions:** "Merge" action added. Merge steps at cluster boundaries (Steps 1.1m, 2m, 3m, 4m). ✓
9. **Parallel Execution Summary:** No "→ Synthesize → analysis.md", no "→ CT-Aggregator → design_critical_review.md", no "→ V-Aggregator → verifier.md", no "→ R-Aggregator → review.md". Memory merge steps shown. ✓
10. **Guard reference at line 148:** "Do NOT produce or reference a combined `verifier.md`" — this is a deliberate prohibition, not a stale reference. Acceptable. ✓

### Researcher — Key Patterns Verified

1. No "Mode 2", "Synthesis", "synthesis mode", or `analysis.md` references. ✓
2. Only focused mode remains with 4 parallel instances. ✓
3. Isolated memory output: `memory/researcher-<focus-area>.mem.md`. ✓
4. Anti-Drift Anchor correctly references isolated memory. ✓

### Planner — Key Patterns Verified

1. Mode Detection: `MODE: REPLAN` from orchestrator + `verification/v-tasks.md` fallback. No `verifier.md` reference. ✓
2. Replan Cross-Referencing Steps 0–5: V sub-agent memories → v-tasks.md → v-tests.md → v-feature.md → prioritize → produce remediation plan. ✓
3. Isolated memory output: `memory/planner.mem.md`. ✓

### R-Security — Key Patterns Verified

1. Pipeline Blocker Override Rule references orchestrator (not R Aggregator). ✓
2. Severity vocabulary: "Blocker / Major / Minor" with explicit "Do not use `Critical` — use `Blocker` instead." ✓
3. Isolated memory template includes `highest_severity` field. ✓

---

## Security Findings

**Standard secrets/PII scan:** CLEAN

Scanned all changed files in `NewAgentsAndPrompts/` for: `password`, `secret`, `api_key`, `apikey`, `token`, `Bearer`, `private_key`, `AWS_`, `AZURE_`, `connection_string`.

All 20 matches are instructional references within agent prompt definitions (e.g., "Flag immediately with `severity: critical`", "run `grep_search` for high-signal patterns (`password`, `secret`, `api_key`, `token`, `Bearer`)"). No hardcoded secrets, API keys, tokens, passwords, or PII found in any file.

---

## Acceptance Criteria Assessment

| AC    | Description                                    | Result   | Notes                                                                                                                                                                                                                                                                          |
| ----- | ---------------------------------------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| AC-1  | Aggregator files removed                       | **PASS** | All 3 contain only deprecation notices (9–10 lines each). No active workflow.                                                                                                                                                                                                  |
| AC-2  | No references to aggregator agents             | **PASS** | Zero active matches. `critical-thinker.agent.md` mentions `ct-aggregator` in its own deprecation notice — acceptable.                                                                                                                                                          |
| AC-3  | No references to removed artifacts             | **PASS** | Zero matches for `analysis.md`, `design_critical_review.md` in active files. `verifier.md` appears once in orchestrator line 148 as a prohibition ("Do NOT produce or reference") — instructional guard, not an artifact path reference.                                       |
| AC-4  | No researcher synthesis mode                   | **PASS** | researcher.agent.md: no "Mode 2", "Synthesis", "synthesis mode", or `analysis.md`. Only focused mode.                                                                                                                                                                          |
| AC-5  | All sub-agents produce isolated memory         | **PASS** | All 21 active agents have `memory/<agent-name>.mem.md` in Outputs, a "Write Isolated Memory" workflow step, and correct Anti-Drift Anchors referencing isolated memory.                                                                                                        |
| AC-6  | Orchestrator memory merge                      | **PASS** | "Merge" action in Memory Lifecycle. Steps 1.1m, 2m, 3m, 4m for cluster merges. Sole writer statement in Global Rule 12.                                                                                                                                                        |
| AC-7  | Orchestrator absorbs CT decision logic         | **PASS** | CT Cluster Decision Flow with severity-based routing. Matches FR-4.1.                                                                                                                                                                                                          |
| AC-8  | Orchestrator absorbs V decision logic          | **PASS** | V Cluster Decision Flow with 9-row decision table. Pattern C replan uses `verification/v-tasks.md`.                                                                                                                                                                            |
| AC-9  | Orchestrator absorbs R decision logic          | **PASS** | R Cluster Decision Flow with R-Security pipeline override, R-Knowledge non-blocking. Safety vocabulary mapping.                                                                                                                                                                |
| AC-10 | Downstream agents read research files directly | **PASS** | spec, designer, planner all list individual research files. No `analysis.md`.                                                                                                                                                                                                  |
| AC-11 | Designer reads CT files directly               | **PASS** | References individual `ct-review/ct-*.md` files. No `design_critical_review.md`.                                                                                                                                                                                               |
| AC-12 | Planner reads V files directly                 | **PASS** | References `verification/v-tasks.md`, `verification/v-tests.md`, `verification/v-feature.md`. No `verifier.md`.                                                                                                                                                                |
| AC-13 | Dispatch patterns updated                      | **PASS** | Pattern A, B, C: no aggregator steps, no combined artifact references. Memory-First Pattern added.                                                                                                                                                                             |
| AC-14 | Memory write safety updated                    | **PASS** | Global Rule 12: no aggregators listed. Orchestrator sole writer. Sub-agents isolated.                                                                                                                                                                                          |
| AC-15 | Documentation structure updated                | **PASS** | `memory/` directory present. No removed artifacts listed.                                                                                                                                                                                                                      |
| AC-16 | Anti-drift anchors updated                     | **PASS** | All 21 active agents have correct Anti-Drift Anchors referencing isolated memory. No unqualified "do NOT write to memory.md" in any Anti-Drift Anchor. (Note: r-quality role description at line 12 has old phrasing — see Issue 1 — but the Anti-Drift Anchor itself passes.) |
| AC-17 | Feature workflow updated                       | **PASS** | No aggregator references, no removed artifacts. Isolated memory model described.                                                                                                                                                                                               |
| AC-18 | Step 1.2 removed                               | **PASS** | No Step 1.2. No synthesis researcher invocation.                                                                                                                                                                                                                               |
| AC-19 | Convention consistency preserved               | **PASS** | Standard section ordering maintained. Operating Rules 1–4 identical across all agents. (Note: Rule 6 varies — see Issue 2 — but AC-19 only requires Rules 1–4 identity.)                                                                                                       |
| AC-20 | Completion contracts unchanged                 | **PASS** | Sub-agent completion contracts remain DONE/ERROR or DONE/NEEDS_REVISION/ERROR. No new fields in completion line.                                                                                                                                                               |

---

## Testing Impact

No runtime tests exist (prompt-file-only project). Structural verification of all AC-1 through AC-20 was performed via text search and manual review. All pass.

**Recommended validation steps before merge:**

1. Run a grep sweep for `aggregator` across `NewAgentsAndPrompts/` excluding the 3 deprecated files and `critical-thinker.agent.md` — expect zero matches.
2. Run a grep sweep for `analysis\.md` across all active files — expect zero matches.
3. Verify `memory.md` Inputs listing is consistent across all CT agents (resolve Issue 2).

---

## Architectural Decisions

No new significant architectural decisions were identified beyond those already documented in the feature specification and design. The implementation faithfully follows the design's branch-merge model, orchestrator-absorbed decision logic, and aggregator deprecation.

The Operating Rule 6 divergence (Issue 2) represents an implicit design decision: whether parallel sub-agents should read shared `memory.md` for broad context or read only targeted upstream memory files. This should be made explicit and consistent. No `decisions.md` entry created since this needs resolution first.

---

## Owners & Next Steps

| Priority | Action                                                 | Owner       | Files                                                                                                                                        |
| -------- | ------------------------------------------------------ | ----------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| Low      | Fix r-quality role description phrasing (Issue 1)      | Implementer | `r-quality.agent.md` line 12                                                                                                                 |
| Low      | Align CT cluster Inputs and Operating Rule 6 (Issue 2) | Implementer | `ct-strategy.agent.md` or `ct-security.agent.md`, `ct-scalability.agent.md`, `ct-maintainability.agent.md`, `orchestrator.agent.md` line 268 |
| Low      | Align documentation-writer Inputs (Issue 3)            | Implementer | `documentation-writer.agent.md`                                                                                                              |

---

## Conformance to .github Instructions

No `.github/instructions/` files exist in the repository. No updates needed.

---

## Checklist / Acceptance

- [x] All 25 files reviewed (3 deprecated, 1 already-deprecated, 21 active)
- [x] Stale reference scans completed (aggregator names, analysis.md, design_critical_review.md, verifier.md, review.md, synthesis mode)
- [x] Isolated memory completeness verified across all 21 active agents (Outputs, workflow step, Anti-Drift Anchor)
- [x] Security routing verified (R-Security pipeline override, Blocker/Major/Minor vocabulary)
- [x] Planner replan mode verified (MODE: REPLAN, 6-step cross-referencing, individual V artifacts)
- [x] Memory-first reading protocol verified across all agents
- [x] Orchestrator specifics verified (Global Rule 12, Documentation Structure, cluster decision flows, NEEDS_REVISION routing, Memory Lifecycle, Parallel Execution Summary)
- [x] Secrets/PII scan: CLEAN
- [x] All AC-1 through AC-20: PASS
- [ ] Issue 1: r-quality role description phrasing — awaiting fix
- [ ] Issue 2: CT cluster Inputs/Rule 6 inconsistency — awaiting alignment decision
- [ ] Issue 3: documentation-writer Inputs — awaiting alignment decision
