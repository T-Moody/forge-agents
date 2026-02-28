# UI Bug Fixes — Implementation Plan

**Feature slug:** `ui-bug-fixes`  
**Planning mode:** Initial  
**Date:** 2026-02-26  
**Agent:** Planner

---

## Success Criteria (from spec-output.yaml)

- All 8 bugs fixed (BUG-001 through BUG-008)
- Existing ~2117 test suite passes without regressions (CR-2)
- No hardcoded strings (CR-3), 44x44px touch targets (CR-4), web projects compile (CR-5)
- Backward compatibility with existing data (CR-1)

---

## Task Index

| #   | Task ID  | Bug IDs          | Title                                                   | Risk | Size     | Wave | Dependencies       |
| --- | -------- | ---------------- | ------------------------------------------------------- | ---- | -------- | ---- | ------------------ |
| 1   | TASK-001 | BUG-001, BUG-008 | Swipe icon sizing + dead CSS cleanup                    | 🟢   | Standard | 1    | —                  |
| 2   | TASK-002 | BUG-005          | Optional skill rating on CreatePlayerPage               | 🟢   | Standard | 1    | —                  |
| 3   | TASK-003 | BUG-007          | Color.Secondary fix on EmptyState/LoadingState          | 🟢   | Standard | 1    | —                  |
| 4   | TASK-004 | BUG-004          | P0: Fix bottom sheet cancel (show form, don't navigate) | 🟡   | Standard | 2    | —                  |
| 5   | TASK-005 | BUG-002          | Template picker in settings template creation           | 🟡   | Standard | 2    | TASK-004           |
| 6   | TASK-006 | BUG-003          | Investigate + fix template creation error               | 🟡   | Standard | 2    | TASK-004, TASK-005 |
| 7   | TASK-007 | BUG-006          | Professional theme Apple HIG + Razor Color.Secondary    | 🟡   | Standard | 3    | —                  |
| 8   | TASK-008 | BUG-006          | 6 remaining themes semantic color standardization       | 🟡   | Standard | 3    | TASK-007           |

---

## Execution Waves

### Wave 1: Independent Low-Risk Fixes (Parallel, max 3 concurrent)

Three independent tasks with zero file overlap. Can execute simultaneously.

```
TASK-001 ─── list-swipe.css (CSS only, 🟢)
TASK-002 ─── CreatePlayerPage.razor.cs + .razor (UI pattern copy, 🟢)
TASK-003 ─── EmptyState.razor + LoadingState.razor (attribute swap, 🟢)
```

**Expected test impact:** 2-4 test updates in CreatePlayerPageTests.cs. Zero test changes for TASK-001 and TASK-003.

### Wave 2: P0 Blocker + Coupled CreateGamePage Fixes (Sequential)

All three tasks modify `CreateGamePage.razor.cs`. Must execute in strict order.

```
TASK-004 ──→ TASK-005 ──→ TASK-006
  (P0 cancel)   (template picker)   (error investigation)
```

**Critical ordering:** TASK-004 fixes cancel semantics. TASK-005 relies on the fixed cancel path. TASK-006 may be resolved by TASK-004/005 changes (the template creation error might stem from the cancel-navigation bug preventing form render).

**Expected test impact:** 5-11 test updates/additions in CreateGamePageTemplateSelectionTests.cs. 1 new integration test in CreateGameHandlerTests.cs.

### Wave 3: Theme Palette Redesign (Sequential)

BUG-006 split into two tasks to stay within 3-file limit.

```
TASK-007 ──→ TASK-008
  (Professional theme    (6 remaining themes
   + Razor fixes)         semantic colors)
```

**Expected test impact:** 5-10 value-specific test updates in ThemeRegistryTests.cs. Invariant tests (BackgroundIsNotPureBlack, Surface≠#FFFFFF, DrawerBackground==Surface) must pass WITHOUT modification.

---

## Dependency Graph

```
TASK-001 ────────────────────────────── (independent)
TASK-002 ────────────────────────────── (independent)
TASK-003 ────────────────────────────── (independent)

TASK-004 ──→ TASK-005 ──→ TASK-006     (CreateGamePage chain)

TASK-007 ──→ TASK-008                   (ThemeRegistry chain)
```

No cross-chain dependencies. Waves 1, 2, and 3 can technically start concurrently (different files), but Wave 2's sequential ordering is internal.

---

## Risk Summary

| Level             | Count  | Tasks                                            |
| ----------------- | ------ | ------------------------------------------------ |
| 🟢 Additive       | 3      | TASK-001, TASK-002, TASK-003                     |
| 🟡 Business Logic | 5      | TASK-004, TASK-005, TASK-006, TASK-007, TASK-008 |
| 🔴 Critical       | 0      | —                                                |
| **Overall**       | **🟡** | Highest risk across all tasks                    |

### Per-File Risk Classification

| File                        | Risk | Tasks                        | Rationale                                        |
| --------------------------- | ---- | ---------------------------- | ------------------------------------------------ |
| `list-swipe.css`            | 🟢   | TASK-001                     | CSS-only, no .NET impact                         |
| `CreatePlayerPage.razor.cs` | 🟡   | TASK-002                     | Modifying form field logic                       |
| `CreatePlayerPage.razor`    | 🟡   | TASK-002                     | Modifying MudNumericField binding                |
| `EmptyState.razor`          | 🟢   | TASK-003                     | Attribute replacement                            |
| `LoadingState.razor`        | 🟢   | TASK-003                     | Attribute replacement                            |
| `CreateGamePage.razor.cs`   | 🟡   | TASK-004, TASK-005, TASK-006 | Cancel/navigation logic, P0                      |
| `ThemeRegistry.cs`          | 🟡   | TASK-007, TASK-008           | 708-line file, hex value changes across 8 themes |
| `ThemePreviewCard.razor`    | 🟢   | TASK-007                     | Attribute replacement                            |
| `AppearancePage.razor`      | 🟢   | TASK-007                     | Attribute replacement                            |
| `CreateGameHandlerTests.cs` | 🟢   | TASK-006                     | New test (additive)                              |

---

## Pre-Mortem Analysis

| Task     | Failure Scenario                                         | Likelihood | Impact | Mitigation                                                                           |
| -------- | -------------------------------------------------------- | ---------- | ------ | ------------------------------------------------------------------------------------ |
| TASK-001 | CSS override doesn't propagate to all pages              | L          | L      | Shared stylesheet; verified by manual inspection on all 5 list pages                 |
| TASK-002 | Null coercion missed in Submit → validation fails        | L          | L      | Validator catches range; default parameter on service provides fallback              |
| TASK-003 | MudText rendering changes with Style vs Color            | L          | L      | No functional change; visual-only                                                    |
| TASK-004 | Cancel fix breaks template selection flow                | L          | H      | Tests cover both cancel and selection paths; revert 2 methods if needed              |
| TASK-005 | Template picker shows empty dialog when no templates     | L          | M      | Guard checks templates exist before invoking dialog                                  |
| TASK-006 | Root cause is deep/complex, not resolved by TASK-004/005 | M          | L      | Defer to Playwright investigation; template creation not on critical tournament path |
| TASK-007 | Hex values break ThemeRegistryTests invariants           | L          | H      | Design specifies invariant-compliant values; run tests after each change             |
| TASK-008 | Missing a theme or wrong hex value                       | L          | M      | Design provides exact from→to table; run tests after each theme                      |

**Overall Risk Level:** Medium — The P0 blocker (TASK-004) has a proven fix pattern from ParticipantsTabContent. The theme redesign (TASK-007/008) has exact hex values from design with invariant compliance verified. BUG-003 investigation (TASK-006) has unknown root cause but is not on critical path.

**Key Assumptions:**

1. BUG-003 root cause is either in the handler/service layer (caught by integration test) or is a symptom of BUG-004's cancel-navigation bug. If neither, Playwright live testing is required. _Affects: TASK-006_
2. The design-output.yaml hex values all satisfy ThemeRegistryTests invariants as documented. _Affects: TASK-007, TASK-008_
3. `Players_SkillRatingHelper` localization key already exists (used by CreateRosterPlayerPage). _Affects: TASK-002_
4. `RosterValidators.ValidateSkillRating` is accessible from CreatePlayerPage. _Affects: TASK-002_
