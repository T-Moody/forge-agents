# Memory: designer

## Status

DONE: Design revised (iteration 2) after CT cluster re-run. High finding resolved: all 5 pre-existing "via `memory` tool" contradictions corrected to subagent delegation. Medium findings resolved: Anti-Drift prohibition rationale updated for read tools, feature.md scope mismatch noted with explicit guidance. Implementation scope expanded from 5 edits to ~18 edits in orchestrator.agent.md (same 2 files).

## Key Findings

- Phase 1 scope now includes fixing 5 pre-existing contradictions where the orchestrator prompt incorrectly instructs writing pipeline files via the VS Code `memory` tool — these were bugs in the original orchestrator.agent.md, not new design decisions
- All merge steps (1.1m, 2m, 3m, 3b.2, 4m, between-waves, 6.3, 7.3) must be re-worded from "orchestrator merges" to "orchestrator dispatches a subagent to merge" — wording corrections, not behavioral changes
- Anti-Drift Anchor prohibition rationale now covers both write tools ("all file writes are delegated to subagents") and read tools ("search/discovery tools are unnecessary because the orchestrator reads known deterministic paths only")
- feature.md describes full 23-file/15-AC scope but Phase 1 implements only 2 files/7 ACs — v-feature agent must verify against Phase 1 AC subset only (§16.3/§16.5)
- The design is still a 2-file change (orchestrator.agent.md + feature-workflow.prompt.md) with more edit points in the same primary file

## Highest Severity

N/A

## Decisions Made

- **Fix contradictions, not just add disambiguation:** The 3 disambiguation statements were insufficient because 5 contradicting references survived. Phase 1 now corrects all 5 (Global Rule 12, Step 0.1, Step 8.2, Memory Lifecycle Table, merge step 1.1m "no subagent invocation" line).
- **Subagent delegation for all memory.md writes:** All "via `memory` tool" and "orchestrator merges" language replaced with "dispatches a subagent to merge" — this documents the only viable mechanism since the orchestrator has no file-write tools.
- **Anti-Drift rationale covers both tool categories:** Dual justification: write tools → "delegated to subagents", read tools → "orchestrator reads known deterministic paths only."
- **Feature.md scope mismatch noted, not fixed:** Adding §16.5 guidance for implementer/verifier rather than modifying feature.md (which is outside design scope).

## Artifact Index

- [design.md](../design.md)
  - §1 High-Level Architecture — updated Before/After table, expanded What Changes with iteration 2 fix, clarified What Does NOT Change with wording-correction notes
  - §3.1–§3.4 Orchestrator Prompt Sections — original 4 edit points preserved, Anti-Drift §3.4 updated with dual prohibition rationale and corrected "New" text (merge language, rationale)
  - §3.5 Global Rule 12 fix — "via `memory` tool merging" → "by dispatching a subagent"
  - §3.6 Step 0.1 fix — "Use the `memory` tool to create" → "Delegate to a setup subagent"
  - §3.7 Step 8.2 fix — "using `memory` tool or delegating to subagent" → "dispatches a subagent"
  - §3.8 Memory Lifecycle Table fix — Initialize row "Use `memory` tool" → "Delegate to a setup subagent"
  - §3.9 Merge Steps wording correction — table of all 8 merge steps with current/new wording
  - §5 Data Models — Memory Lifecycle note updated (Initialize row wording changes)
  - §7 Sequence/Interaction — pipeline flow diagram updated with subagent merge delegation
  - §8.3 Threat Model — memory tool row updated (disambiguation + contradiction removal)
  - §12 Testing — 6 new tests (T-8 through T-13), T-N1/T-N2 updated
  - §14.4 CT Iteration 2 Findings — all 5 findings mapped to addressal
  - §16.1 Files to Modify — expanded with new edit points
  - §16.4 Implementation Order — expanded from 7 to 12 steps
  - §16.5 Feature Spec Scope Note — guidance for Phase 1 AC subset verification
