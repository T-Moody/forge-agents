# Adversarial Review: design — pragmatic-verifier

## Review Metadata

- **Perspective:** pragmatic-verifier
- **Risk Level:** 🟡
- **Round:** 2
- **Run ID:** 2026-03-05T12:00:00Z

## Round 1 Finding Disposition

| R1 Finding                                            | Severity | Status        | Resolution                                                                                                                                                                                                                  |
| ----------------------------------------------------- | -------- | ------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| PV-A1: Orchestrator baseline 353 not 539              | Major    | **RETRACTED** | My R1 measurement was incorrect. PowerShell `Measure-Object -Line` counts CRLF terminators; the orchestrator uses LF-only line endings. Splitting on LF confirms 539 content lines. Designer's 539/550 baseline is correct. |
| PV-C1: FR-2.2 per-agent docs missing                  | Major    | **Resolved**  | All 8 subagent files now include FR-2.2 in `addresses_frs`. `adversarial-reviewer.agent.md` and `knowledge-agent.agent.md` added to inventory. `per_issue_summary` Issue #2 lists all 9 files.                              |
| PV-C2: Dual never-skip-steps gap                      | Major    | **Resolved**  | Both line 29 (Role & Purpose) and line 539 (anti-drift anchor) amendments documented with same conditional language and minimum step set {Step 0, Step 7, Step 9} per D-14.                                                 |
| PV-C3: "Default to autonomous" count 3→2              | Minor    | **Resolved**  | D-4 corrected to "2 times in lines 105 and 109, with related autonomous-default language at line 24."                                                                                                                       |
| PV-C4: allowFreeformInput per-question not per-option | Minor    | **Resolved**  | Design.md Sequence/Issue #5 clarified: "enables freeform text for all options on this question; spec agent processes freeform only when 'modify' is selected." D-7 updated.                                                 |
| PV-C5: Fast-track + autonomous edge case              | Minor    | **Resolved**  | EC-14 added to Failure & Recovery table. Sequence/Issue #8 documents: "In autonomous mode (EC-14): works from initial-request.md only, lower confidence."                                                                   |
| PV-C6: Step 4 input description for fast-track        | Minor    | **Resolved**  | Sequence/Issue #8 notes: "In fast-track mode, Step 4 input is initial-request.md (NOT spec/design artifacts)."                                                                                                              |

## Security Analysis

**Category Verdict:** approve

No security findings. Assessment through pragmatic-verifier lens (logical errors creating security holes):

- **D-14 (minimum step set):** The immutable constraint {Step 0, Step 7, Step 9} placed in both line 29 and line 539 is a sound defense-in-depth measure. No logical path exists to bypass Step 7 (code review) given this constraint. Step 9 (commit) ensures evidence is captured.
- **fetch_webpage (D-5):** Interactive-mode-only restriction is naturally enforced by VS Code's per-invocation approval. Scope documentation is clear per agent. No logical gap that would allow autonomous-mode web fetching.
- **DB archive (D-9):** Archive rename is bounded to a single deterministic file path. No path traversal possible since the rename target is computed from the original filename + timestamp suffix.
- **APPROVAL_MODE priority (DR-2):** Relocating to orchestrator Step 0 is architecturally sound — the enforcement point is where the mode is actually selected. No logical gap between declaration and enforcement.

## Architecture Analysis

**Category Verdict:** approve

No architecture findings requiring revision. Assessment through pragmatic-verifier lens (naming/path accuracy, consumer contracts):

- **PV-A1 RETRACTED:** The 539/550 baseline is correct. My R1 measurement used `Measure-Object -Line` which counts CRLF line terminators; this file uses LF-only endings, yielding 353 (an artifact of the counting method, not the file). Verified via `[System.IO.File]::ReadAllText().Split(LF)` returning 540 elements (539 content lines + 1 trailing). D-1 (Direction C) rationale is valid — the orchestrator at 539/550 genuinely cannot absorb all changes inline.
- **DR-2:** Clean resolution of FR-2.4/FR-8.6 contradiction. APPROVAL_MODE priority now in orchestrator Step 0 (enforcement point) rather than feature-workflow.prompt.md (declaration point). Architecture is more consistent — prompt defines WHAT runs, orchestrator defines HOW.
- **File inventory completeness:** 15 files (1 new + 14 modified) with explicit `addresses_frs` lists providing traceability. All 38 sub-requirements mapped. Wave decomposition (4 waves) follows dependency ordering correctly — foundation schemas before agent modifications before reference docs before prompts.
- **Consumer contracts:** All file inventory entries include estimated lines, risk rating, and change description. Sufficient for the planner to generate task definitions.

## Correctness Analysis

**Category Verdict:** approve

Two new Minor findings identified. No Major or Critical issues remain.

### Finding C-1: D-11 line budget arithmetic — per-change estimates sum to +15, not +11

- **Severity:** Minor
- **Description:** D-11 lists per-change line estimates: (a) -6, (b) -2, (c) +8, (d) +5, (e) +6, (f) +2, (g) +1, (h) +1. Sum: -8 + 23 = **+15**. The design states "Net: ~+11 lines → 550/550." The discrepancy appears to be that (a) and (e) are the same approval-mode change (remove ~6 lines then add ~6 lines = net 0), but the remaining items (b)+(c)+(d)+(f)+(g)+(h) = -2+8+5+2+1+1 = **+15** regardless of how (a)/(e) are counted. The likely explanation: R1's ~+11 estimate omitted R2 additions — items (f), (g), (h) were expanded or added to resolve S-1, PV-C2, and A-1, adding +4 lines unaccounted for in the total. At +15, the result would be 539+15 = **554/550**, 4 lines over budget.
- **Affected artifacts:** design-output.yaml (D-11 rationale, orchestrator.agent.md entry net change claim)
- **Recommendation:** No design revision needed. The design already documents a sound contingency: "if implementation reveals more lines needed, DB archive procedure (~8 lines) is first extraction candidate." The implementer will discover the actual count and apply extraction if needed. The ~+11 estimate should be corrected to ~+15 for accuracy, but this does not change the design's direction or implementation approach.
- **Evidence:** D-11 items: (a) -6, (b) -2, (c) +8, (d) +5, (e) +6, (f) +2, (g) +1, (h) +1. Arithmetic: -6-2+8+5+6+2+1+1 = 15. Alternatively treating (a)+(e) as net 0: -2+8+5+2+1+1 = 15. Either way, +15 not +11.

### Finding C-2: AC-21 line count verification method unreliable on Windows

- **Severity:** Minor
- **Description:** The testing strategy specifies `wc -l` for AC-21 (orchestrator ≤550 lines). On Windows (the workspace OS), the common PowerShell equivalent `Measure-Object -Line` counts CRLF terminators. The orchestrator file uses LF-only line endings, causing `Measure-Object -Line` to return 353 instead of 539. This is the exact measurement error that produced my R1 PV-A1 false alarm. If the verifier uses `Measure-Object -Line`, AC-21 would pass (353 ≤ 550) for the wrong reason, masking a genuine budget violation.
- **Affected artifacts:** design.md (Testing Strategy, AC-21 method)
- **Recommendation:** The testing strategy should specify a line-counting method robust to mixed line endings on Windows. Suitable alternatives: `(Get-Content file | Measure-Object -Line).Lines` with `-Encoding UTF8` or `[System.IO.File]::ReadAllLines(path).Count`. Alternatively, note that `wc -l` requires Git Bash or WSL on Windows. This is a documentation improvement for the implementer/verifier, not a design change.
- **Evidence:** `(Get-Content "NewAgents\.github\agents\orchestrator.agent.md" | Measure-Object -Line).Lines` returns 353 on this workspace. `[System.IO.File]::ReadAllText(file).Split(LF).Count` returns 540 (539 content + 1 trailing). The file's line ending format is LF-only.

## Summary

All 3 Major findings from Round 1 are fully resolved. PV-A1 is retracted — my measurement methodology was incorrect; the orchestrator is genuinely 539 lines. PV-C1 (FR-2.2 coverage) is resolved with all 8 subagent files now in the inventory. PV-C2 (dual never-skip-steps) is resolved with both line 29 and line 539 amended to the same conditional language with minimum step set. All 4 Minor findings are also addressed. Two new Minor findings identify an arithmetic discrepancy in D-11's budget math (+15 not +11) and a Windows-specific line counting pitfall for AC-21 verification. Neither finding blocks implementation — the design has appropriate contingency plans and the implementer will discover actual line counts during Task 4. The R2 design is comprehensive, implementable, and internally consistent.
