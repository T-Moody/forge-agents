# Memory: designer

## Status

DONE (v4 — Adversarial Review Response): design.md revised from v3 to v4 addressing all 17 findings from 3-model adversarial review (6 Critical, 11 High). Key structural changes: (1) anvil_checks schema expanded with verdict/severity/round/run_id, (2) WAL mode mandated, (3) Deviation Records added for spec divergences, (4) multi-model routing acknowledged as unverified, (5) git tag baseline replaces impossible re-run approach, (6) evidence gates corrected with verdict/round filters, (7) YAML verdict output for reviewers, (8) evidence bundle assembly step added, (9) Implementer self-fix loop + git staging, (10) Verifier Tier 4 operational readiness, (11) FR-4.7 revert assigned to Implementer, (12) lightweight pushback at Step 0, (13) auto-commit as Step 9.

## Key Findings

- C5 (baseline cross-check) was physically impossible in v3 — Verifier sees post-implementation files; can't re-run Tier 1 on pre-change state. Resolved with git tag approach.
- C4 (multi-model routing) is honestly unverified — no evidence `.agent.md` model directives work. Perspective-diverse prompts (security/architecture/correctness) provide genuine analytical diversity as fallback.
- C3 (spec deviations) required formal Deviation Records: DR-1 (orchestrator `run_in_terminal` vs FR-1.6) and DR-2 (always-3 reviewers vs FR-5.3/CR-4). Both are intentional and well-justified.
- H1/H5 synergy: YAML verdict summary + review_focus parameter together create structured, perspective-diverse review output that supports both automated routing and fallback scenarios.
- Orchestrator now has 5 responsibilities (v4: added pipeline initialization — SQLite/WAL/git hygiene/pushback)

## Highest Severity

N/A

## Decisions Made

### v1 Decisions (retained)

- Selected synthesized A+D+B architecture — Confidence: High, Risk: 🟡
- 9 agents with zero-merge typed YAML memory model — Confidence: High, Risk: 🟢
- Always-on adversarial review — Confidence: High, Risk: 🟢

### CT Revision Decisions (v2, retained with modifications)

- Finding #1: Pipeline manifest eliminated — Confidence: High (retained through v4)
- Finding #2: Schema validation = prompt-level structural guidance — Confidence: High (retained through v4)
- Finding #3: YAML-primary verification → **SUPERSEDED in v3** by SQL-primary via Anvil `anvil_checks` schema
- Findings #4-#8: Retained through v4

### v3 Corrections (retained)

- **Correction #1 (SQLite First-Class):** Retained and expanded in v4 — Confidence: High
- **Correction #2 (Always-3 Reviewers):** Retained and enhanced with perspective diversity — Confidence: High
- **Data Storage Strategy:** Retained and expanded with review-verdicts YAML row — Confidence: High

### v4 Decisions (Adversarial Review Response)

- **C1 (Schema expansion):** Added verdict/severity/round/run_id to anvil_checks + output_snippet length constraint + composite indexes — Confidence: High
- **C2 (WAL mandatory):** PRAGMA journal_mode=WAL + busy_timeout=5000 in centralized Step 0 init — Confidence: High
- **C3 (Deviation Records):** DR-1 (run_in_terminal) and DR-2 (always-3 reviewers) with rationale and spec reconciliation guidance — Confidence: High
- **C4 (Multi-model unverified):** Honestly acknowledged; risk downgraded 🟢→🟡; perspective-diverse prompts as primary fallback — Confidence: Medium (assumption unverified)
- **C5 (Baseline via git tag):** pipeline-baseline-{run_id} tag before implementation; Verifier uses `git show` — Confidence: High
- **C6 (Evidence gates corrected):** All queries filter on verdict/round/run_id/check_name; 6 gate types — Confidence: High
- **H1 (YAML verdict):** review-verdicts/<scope>.yaml with structured fields — Confidence: High
- **H2 (Evidence Bundle):** Step 8b non-blocking assembly by Knowledge Agent — Confidence: High
- **H3 (Self-fix loop):** Implementer max 2 self-fix attempts before returning — Confidence: High
- **H4 (Early pushback):** Lightweight evaluation in Step 0, Blocker = halt — Confidence: High
- **H5 (review_focus):** security/architecture/correctness parameter for perspective diversity — Confidence: High
- **H8 (Git staging):** git add -A after Implementer self-fix — Confidence: High
- **H9 (FR-4.7 revert):** Implementer in revert mode via git checkout baseline tag — Confidence: High
- **H10 (Anvil features):** Git hygiene + auto-commit added; session recall intentionally omitted (accepted capability regression) — Confidence: High
- **H11 (Tier 4):** Operational Readiness for Large tasks — observability, degradation, secrets — Confidence: High

## Artifact Index

- [design.md](../design.md) (v4 — Adversarial Review Response, ~2270 lines)
  - §Title & Summary — v2 + v3 + v4 revision notes
  - §Context & Inputs — unchanged from v2
  - §Decision 1: Architecture Selection — unchanged from v3
  - §Decision 2: Agent Inventory — Orchestrator: 5 responsibilities + run_in_terminal constraint (DR-1); Verifier: Tier 4 added, baseline via git tag, corrected SQL queries; Adversarial Reviewer: YAML verdict, review_focus parameter, perspective-diverse dispatch
  - §Decision 3: Pipeline Structure — Step 0 expanded (SQLite init, git hygiene, pushback); Step 3b/7 perspective-diverse; Step 5 self-fix + git add; Step 6 Tier 4 + corrected SQL; Steps 8b/9 NEW; Gate Points v4-corrected; Mermaid diagram updated; Orchestrator Decision Table v4
  - §Decision 4: Communication & State — unchanged
  - §Decision 5: Memory System — unchanged
  - §Decision 6: Verification Architecture — Schema expanded (C1); WAL mandatory (C2); Evidence gates corrected (C6); Baseline via git tag (C5); Revert via Implementer (H9); task_id convention (H6)
  - §Decision 7: Adversarial Review — Model Routing Fallback rewritten (C4); review_focus (H5)
  - §Decision 8: Risk & Escalation — unchanged from v3
  - §Decision 9: Approval Mode — unchanged
  - §Decision 10: File & Output Structure — review-verdicts/ added
  - §Security — unchanged from v3
  - §Failure & Recovery — Model routing row updated; Graceful Degradation updated
  - §Non-functional — Model routing fallback updated
  - §Sequence/Interaction Notes — Corrected SQL queries; git tag baseline; self-fix; evidence bundle; auto-commit
  - §Orchestrator Decision Table — v4 with SQL verdict checks, pushback, revert, auto-commit
  - §Data Storage Strategy — v4; second schema copy updated; review-verdicts YAML row; operational notes WAL mandatory; telemetry run_id column
  - §Deviation Records — NEW (C3): DR-1 (run_in_terminal vs FR-1.6), DR-2 (always-3 vs FR-5.3/CR-4)
  - §v4 Revision Log — NEW: All 17 findings with resolution and sections changed
  - §Data Storage Strategy (NEW) — SQLite vs YAML vs Markdown decision matrix, design principles, full SQL schema details
  - §v3 Revision Log (NEW) — Correction #1 (SQLite first-class) + Correction #2 (always-3 reviewers)
  - §CT Revision Log — Finding #8 Mitigation 3 updated for SQL-primary
