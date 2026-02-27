# Adversarial Design Review — Round 2, Reviewer 3

**Model persona:** claude-opus-4.6
**Focus lens:** Scalability / Maintainability
**Design version reviewed:** v4 (2277 lines)
**Round 1 findings:** 15 (1 Critical, 5 High, 7 Medium, 2 Low)

---

## Final Verdict: APPROVE

The v4 revision substantively addresses 9 of 15 Round 1 findings, partially addresses 4, and leaves 2 unresolved at Medium/Low severity. The most impactful fixes — Evidence Bundle assembly (H2), Implementer self-fix loop (H3), perspective-diverse review focus (H5), FR-4.7 revert assignment (H9), and Tier 4 operational readiness (H11) — are well-designed and correctly integrated into the pipeline structure. Two new medium-severity findings (SQL schema inconsistency, non-blocking Evidence Bundle) and two low-severity observations were identified. No Critical or High-severity issues remain.

---

## Round 1 Finding Verification

### Finding 1 (Critical): Multi-model routing is unverified

**Status: RESOLVED**

**Evidence:** §Decision 7 (Model Routing Fallback) rewritten with honest acknowledgment: "The ability to route subagent dispatches to specific LLM models via `.agent.md` configuration is **unverified**." Risk downgraded from 🟢 to 🟡. The primary fallback is perspective-diverse prompts (security/architecture/correctness focus), not same-prompt different-model. The `review_focus` parameter ensures genuine analytical diversity even when model diversity is unavailable. This is a substantive fallback, not a hand-wave.

---

### Finding 2 (High): Complexity relocated, not reduced — mega-agents

**Status: PARTIALLY_RESOLVED**

**Evidence:** The design continues to use unified Verifier (absorbing 4 Forge agents) and unified Adversarial Reviewer (absorbing 7). However:

- §Decision 2 explicitly acknowledges: "The Verifier is the most complex agent — complexity is managed by per-task dispatch (bounded context per invocation) rather than claiming 'bounded scope.'"
- Per-task Verifier dispatch bounds context to ~8 tool calls (CT revision, preserved).
- Adversarial Reviewer now has bounded focus per instance via `review_focus`.
- The suggestion to split the Verifier into Build/Test + Acceptance agents was not adopted.

**Remaining concern:** The honest acknowledgment of tradeoff is good. The mitigation (per-task dispatch) is genuine. This is acceptable as a documented tradeoff — not a blocking issue.

---

### Finding 3 (High): Anvil's tight verify-fix loop lost

**Status: RESOLVED**

**Evidence:** §Pipeline Overview Step 5 now includes: "Self-fix loop: run IDE diagnostics + build + tests; if fail, fix immediately (max 2 self-fix attempts within Implementer context before returning)." This mirrors Anvil's pattern: the same agent that wrote the code fixes it immediately with full context. The independent Verifier (Step 6) runs afterward for cross-checking. This is exactly the suggested fix from Round 1.

---

### Finding 4 (High): Evidence Bundle dropped

**Status: RESOLVED**

**Evidence:** Step 8b added: "Evidence Bundle Assembly (Knowledge Agent sub-step)" producing `evidence-bundle.md` with: (a) overall confidence rating with Anvil's precise definitions, (b) aggregated verification summary, (c) adversarial review summary, (d) rollback command, (e) blast radius, (f) known issues. This directly addresses the missing user-facing trust deliverable. (Note: a new concern about its non-blocking nature is raised below as N1.)

---

### Finding 5 (High): Tri-format storage over-engineered

**Status: PARTIALLY_RESOLVED**

**Evidence:** §Data Storage Strategy (new v4 section) provides a detailed format decision matrix with 12+ data types mapped to their format with rationale. The design principle "No format serves all needs" is now explicit. Each format has a defined lane: SQLite for evidence/telemetry, YAML for agent I/O, Markdown for human docs. The justification is stronger than v3.

**Remaining concern:** The complexity tax still exists — 3 formats instead of 2, and agents must be literate in 2+ formats. However, the justification is now well-documented as a conscious engineering tradeoff. Not blocking.

---

### Finding 6 (High): Pushback fires too late

**Status: RESOLVED**

**Evidence:** Step 0 now includes: "Lightweight pushback evaluation: Orchestrator evaluates initial-request.md for obvious scope/conflict issues BEFORE dispatching researchers. If Blocker-severity concern found → halt in both interactive and autonomous mode. Major/Minor → log and proceed." This adds a coarse filter at Step 0 (before researchers) while preserving the fine filter at Step 2 (Spec agent with full research context). The two-tier pushback approach is well-designed.

---

### Finding 7 (Medium): Justification scoring is self-assessment

**Status: UNRESOLVED**

**Evidence:** §Decision 8 (Justification Scoring Mechanism) is unchanged from v3. Confidence levels are still self-assessed by the producing agent. No minimum requirement for alternatives considered. No independent validation. No consequences for low-confidence decisions (a Low-confidence decision proceeds identically to High-confidence).

**Assessment:** This was not addressed in v4. However, the Adversarial Design Review (Step 3b) implicitly provides some challenge to designer decisions — reviewers can flag poorly justified decisions. The `review_focus: correctness` reviewer is prompted to check "spec compliance" which includes decision quality. This is an incomplete but functional mitigation via the review process rather than the scoring mechanism itself.

---

### Finding 8 (Medium): Git hygiene, recall, auto-commit dropped

**Status: RESOLVED**

**Evidence:**

- **Git hygiene:** Added to Step 0 — `git status --porcelain`, `git rev-parse --abbrev-ref HEAD`.
- **Auto-commit:** Added as Step 9 with structured commit message and commit SHA recording.
- **Session recall:** "Intentionally omitted" per H10 resolution — VS Code `store_memory` provides broad patterns but not per-file regression history. Acknowledged as "known capability regression."

All three items have been addressed or explicitly acknowledged with justification for omission.

---

### Finding 9 (Medium): Orchestrator `run_in_terminal` reversal

**Status: RESOLVED**

**Evidence:** §Deviation Records DR-1 explicitly acknowledges the deviation from FR-1.6, provides rationale (independent evidence verification, SQLite init, git operations), applies concrete constraints ("`run_in_terminal` ONLY for: SQLite read queries, SQLite DDL, Git read operations, Git staging/commit"), and identifies spec reconciliation needed. The constraint is well-scoped and documented in the orchestrator's anti-drift anchor.

---

### Finding 10 (Medium): Always-3 reviewers wastes dispatches

**Status: RESOLVED**

**Evidence:** §Deviation Records DR-2 acknowledges the spec divergence from FR-5.3 and CR-4. Provides 4 justification points: simplicity, consistent quality, low marginal cost (parallel = wall-clock ≈ 1 reviewer), Anvil alignment. Calculates tradeoff: "4 additional dispatches per pipeline run (26 vs 22) = ~18% more dispatches for 100% consistent review coverage." Identifies spec reconciliation needed.

---

### Finding 11 (Medium): Perspective diversity lost (model ≠ perspective)

**Status: RESOLVED**

**Evidence:** §Decision 2 (Adversarial Reviewer) now includes `review_focus` dispatch parameter with 3 distinct values: `security` (injection, auth, data exposure), `architecture` (coupling, scalability, boundaries), `correctness` (edge cases, logic errors, spec compliance). Each of the 3 parallel reviewers gets a different focus. "ALWAYS dispatched as 3 parallel instances with distinct `review_focus` values." This directly combines model diversity with perspective diversity.

---

### Finding 12 (Medium): YAML layer cost vs. benefit

**Status: PARTIALLY_RESOLVED**

**Evidence:** §Data Storage Strategy provides a principled justification: "Agent narrative output goes in YAML. Requirements, design decisions, implementation plans — anything with rationale, descriptions, or nested structure." The honest limitation about prompt-level-only validation is preserved. The format decision matrix makes the cost-benefit tradeoff explicit.

**Remaining concern:** The ROI question stands as a philosophical disagreement, not a design flaw. The design's position is internally consistent and well-documented.

---

### Finding 13 (Medium): FR-4.7 revert still unassigned

**Status: RESOLVED**

**Evidence:** §Decision 6 (Verification-Replan Loop): "After 2 failed fix attempts for a specific verification check, the **Implementer** is re-dispatched in **revert mode** with parameters: `{mode: 'revert', files_to_revert: [...], baseline_tag: 'pipeline-baseline-{run_id}'}`. The Implementer executes `git checkout pipeline-baseline-{run_id} -- {files}` via `run_in_terminal`." Clear agent assignment with specific parameters and command. Justified: "The Implementer is the only agent with both the tools and permissions to revert."

---

### Finding 14 (Low): "Evidence gating" overstates mechanism

**Status: PARTIALLY_RESOLVED**

**Evidence:** The "Honest limitation" callout in §Decision 6 remains. The v4 corrections improve the mechanism (run_id/round/verdict filters prevent stale records and cross-run contamination). However, the term "evidence gating" is still used throughout without qualification. The suggestion to rename or add parenthetical was not adopted.

**Assessment:** The mechanism itself is now stronger (corrected queries). The terminology concern is cosmetic.

---

### Finding 15 (Low): Forge capabilities lost

**Status: PARTIALLY_RESOLVED**

**Evidence:**

- Post-mortem: Still folded into Knowledge Agent (unchanged). Knowledge capture ≠ retrospective analysis, but the scope expansion is acknowledged.
- Context compression: Targeted context mechanism (Planner's `relevant_context`) provides a different but functional approach.
- Documentation writer: Still merged into Implementer via `task_type: documentation`.
- NEEDS_REVISION for Designer: The Designer returns DONE | ERROR only, but design revision is triggered by Adversarial Reviewer verdicts routed through the orchestrator — architecturally sound, the Designer doesn't need its own NEEDS_REVISION.

**Assessment:** Acceptable. Each capability loss is either mitigated or justified.

---

## Round 1 Summary

| #   | Finding                                  | Severity | Status             | v4 Fix                                                           |
| --- | ---------------------------------------- | -------- | ------------------ | ---------------------------------------------------------------- |
| 1   | Multi-model routing unverified           | Critical | RESOLVED           | C4: honest acknowledgment + perspective-diverse fallback         |
| 2   | Complexity relocated, not reduced        | High     | PARTIALLY_RESOLVED | Acknowledged as tradeoff; per-task dispatch bounds context       |
| 3   | Tight verify-fix loop lost               | High     | RESOLVED           | H3: Implementer self-fix loop (max 2 attempts)                   |
| 4   | Evidence Bundle dropped                  | High     | RESOLVED           | H2: Step 8b Evidence Bundle Assembly                             |
| 5   | Tri-format storage over-engineered       | High     | PARTIALLY_RESOLVED | Data Storage Strategy section with principled justification      |
| 6   | Pushback fires too late                  | High     | RESOLVED           | H4: Lightweight pushback in Step 0                               |
| 7   | Justification scoring self-assessment    | Medium   | UNRESOLVED         | Not addressed; implicitly mitigated by adversarial review        |
| 8   | Git hygiene, recall, auto-commit dropped | Medium   | RESOLVED           | H10: Git hygiene + auto-commit added; recall justified omission  |
| 9   | Orchestrator `run_in_terminal` reversal  | Medium   | RESOLVED           | DR-1: Deviation record with constraints                          |
| 10  | Always-3 reviewers wastes dispatches     | Medium   | RESOLVED           | DR-2: Deviation record with cost justification                   |
| 11  | Perspective diversity lost               | Medium   | RESOLVED           | H5: `review_focus` parameter (security/architecture/correctness) |
| 12  | YAML layer cost vs. benefit              | Medium   | PARTIALLY_RESOLVED | Better justified; philosophical disagreement, not design flaw    |
| 13  | FR-4.7 revert unassigned                 | Medium   | RESOLVED           | H9: Implementer revert mode assigned                             |
| 14  | Evidence gating overstates mechanism     | Low      | PARTIALLY_RESOLVED | Mechanism improved; terminology unchanged                        |
| 15  | Forge capabilities lost                  | Low      | PARTIALLY_RESOLVED | H10/H2 address some; remainder justified                         |

**Resolved: 9 | Partially Resolved: 4 | Unresolved: 2**

---

## NEW Findings (v4-Introduced)

### [N1]: Evidence Bundle Assembly Is Non-Blocking — Critical Deliverable Can Silently Fail

- **Severity:** Medium
- **Area:** §Decision 3 (Step 8b), §Decision 2 (Knowledge Agent)
- **Issue:** Step 8b produces `evidence-bundle.md` — the primary user-facing trust deliverable that was added to address Round 1 Finding 4. However, Step 8b inherits the Knowledge Agent's "non-blocking" property: "failure does not halt pipeline." If the Knowledge Agent encounters an error (context overflow, tool failure), the user receives no evidence bundle, no confidence rating, no rollback command, and no blast radius analysis. The pipeline reports DONE without its most important user-facing output.
- **Where:** Pipeline Overview Step 8b ("Non-blocking: failure does not halt pipeline (same as Step 8)")
- **Likelihood:** Low (Knowledge Agent failure is uncommon)
- **Impact:** Medium — User loses the primary proof-of-quality deliverable with no indication it's missing.
- **Assumption at risk:** That the Knowledge Agent will reliably succeed. If it fails, the Evidence Bundle fix for H2 becomes a no-op.
- **Recommendation:** Either (a) make the Evidence Bundle assembly a separate blocking micro-step after the non-blocking Knowledge Capture, or (b) have the orchestrator generate a minimal evidence bundle (confidence + rollback command + pass counts from SQL queries) as a fallback when Step 8b fails. The orchestrator already has `run_in_terminal` for SQL queries — it could produce a basic bundle with a few SELECT queries.

---

### [N2]: SQL Schema Inconsistency — `output_snippet` Length Limit 500 vs 2000

- **Severity:** Medium
- **Area:** §Decision 6 (SQL Schema) vs §Data Storage Strategy (SQLite Schema Details)
- **Issue:** The `anvil_checks` schema appears in two locations with conflicting `output_snippet` length constraints:
  - §Decision 6 (line ~684): `output_snippet TEXT CHECK(LENGTH(output_snippet) <= 500)` — noted as "matching Anvil truncation rule"
  - §Data Storage Strategy (line ~2058): `output_snippet TEXT CHECK(length(output_snippet) <= 2000)`
- These are contradictory. The v4 revision note (C1) says "output_snippet 500-char length constraint (matching Anvil truncation rule)" — confirming 500 was intended.
- **Where:** design.md lines 684 and 2058
- **Likelihood:** High (implementer will encounter this inconsistency)
- **Impact:** Medium — Whichever schema is used in the CREATE TABLE, the other location's documentation is wrong. If 2000 is used but agents truncate at 500 (per v4 C1 guidance), data is consistent but the schema is over-permissive. If 500 is used but agents write longer snippets (per the 2000 limit), INSERTs fail.
- **Assumption at risk:** That the design document is internally consistent for schema definitions.
- **Recommendation:** Reconcile to 500 (matching Anvil) in both locations. The Data Storage Strategy section's 2000-char version should be corrected.

---

### [N3]: Rollback Command Invalid When Auto-Commit Is Skipped

- **Severity:** Low
- **Area:** §Decision 3 (Steps 8b and 9)
- **Issue:** The Evidence Bundle includes rollback command `git revert --no-commit pipeline-baseline-{run_id}..HEAD`. But Step 9 skips auto-commit when Confidence is Low. `git revert` operates on committed ranges — if there's no commit (Confidence: Low), the rollback command is invalid. The correct rollback for uncommitted changes would be `git checkout pipeline-baseline-{run_id}` or `git reset --hard pipeline-baseline-{run_id}`.
- **Where:** Pipeline Overview Step 8b (line 400) vs Step 9 (line 407)
- **Likelihood:** Medium (Low-confidence runs are the scenario where rollback is most needed)
- **Impact:** Low — User would quickly discover the command doesn't work and use `git checkout` instead.
- **Assumption at risk:** That the Evidence Bundle rollback command is always valid.
- **Recommendation:** Evidence Bundle should adapt rollback command: if auto-commit occurred, use `git revert`; if not, use `git checkout pipeline-baseline-{run_id}`.

---

### [N4]: Tier 4 Secrets Scan Limited to Large Tasks Only

- **Severity:** Low
- **Area:** §Decision 2 (Verifier detail), §Decision 3 (Step 6)
- **Issue:** Tier 4 Operational Readiness (which includes "no hardcoded secrets" scanning) only runs for Large tasks. A Standard task modifying a configuration file (classified 🟢/🟡) could accidentally introduce a hardcoded API key or credential without triggering a secrets scan. Tier 2 linting may catch this if the project has lint rules for secrets, but there's no guarantee.
- **Where:** Verifier cascade Tier 4 condition: "Large tasks only"
- **Likelihood:** Low (most secrets issues are in credential-handling code, which would be 🔴)
- **Impact:** Low — Adversarial code review (Step 7) with `review_focus: security` would likely catch exposed secrets even without Tier 4.
- **Assumption at risk:** That 🔴 classification reliably captures all files where secrets could be introduced.
- **Recommendation:** Consider promoting the secrets-scan check (`check_name='readiness-secrets'`) to Tier 2 for all tasks, leaving observability and degradation checks in Tier 4 for Large tasks only.

---

## New Findings Summary

| #   | Finding                                                        | Severity | Category                    |
| --- | -------------------------------------------------------------- | -------- | --------------------------- |
| N1  | Evidence Bundle non-blocking — can silently fail               | Medium   | Scalability/Maintainability |
| N2  | SQL schema `output_snippet` length inconsistency (500 vs 2000) | Medium   | Maintainability             |
| N3  | Rollback command invalid when auto-commit skipped              | Low      | Correctness                 |
| N4  | Tier 4 secrets scan limited to Large tasks                     | Low      | Security/Scalability        |

---

## Overall Assessment

The v4 revision represents a thorough and thoughtful response to Round 1 findings. The designer addressed 9 of 15 findings completely and provided principled justification for the 4 partially resolved items. The two unresolved findings (justification scoring self-assessment at Medium, Forge capabilities lost at Low) are not blocking — one is implicitly mitigated by the adversarial review process, the other is a collection of minor capability regressions that are individually acceptable.

The 4 new findings are at Medium and Low severity. N2 (schema inconsistency) is the most actionable — a simple document fix. N1 (non-blocking Evidence Bundle) is the most architecturally interesting but has low likelihood of triggering.

The design's growth from ~1800 to ~2277 lines is notable but primarily driven by honest documentation: deviation records, expanded revision logs, and the Data Storage Strategy section. The accumulated revision logs (CT, v3, v4) account for ~400+ lines that could be extracted to a separate `revision-history.md` in future to improve readability, but this is a cosmetic concern.

**Highest remaining severity:** Medium (N1, N2, and unresolved Finding 7)
**Blocking issues:** None
