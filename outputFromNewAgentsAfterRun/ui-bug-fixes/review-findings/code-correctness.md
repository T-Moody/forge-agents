# Adversarial Review: code — claude-opus-4.6

## Review Metadata

- **Review Focus:** correctness
- **Risk Level:** 🟡
- **Round:** 1
- **Run ID:** 2026-02-26T14:30:00Z

## Overall Verdict

**Verdict:** approve
**Confidence:** High

## Findings

### Finding 1: Contradictory tests for saveAsTemplate EventId guard

- **Severity:** Major
- **Category:** correctness
- **Description:** Two tests in `CreateGamePageTemplateSelectionTests` document contradictory expectations for whether `saveAsTemplate` is guarded by `EventId.HasValue`. `Submit_SaveAsTemplateGuardedByEventContext` (line ~418) asserts `effectiveSaveAsTemplate = saveAsTemplate && eventId.HasValue` — false when `eventId=null`. `Submit_SaveAsTemplate_NotGuardedByEventId` (line ~768) asserts `effectiveSaveAsTemplate = saveAsTemplate` — true when `eventId=null`. Both tests pass because they test local variable logic, not production code. Production code at `CreateGamePage.razor.cs` line 412 passes `_saveAsTemplate` directly without `EventId.HasValue` guard, matching the newer test.
- **Affected artifacts:** `tests/TourneyPal.Tests/Features/Games/CreateGame/CreateGamePageTemplateSelectionTests.cs` lines 418–428 vs 768–776
- **Recommendation:** Remove or update `Submit_SaveAsTemplateGuardedByEventContext` to match the current production behavior (no EventId guard). The stale test documents a superseded specification and could mislead future developers.
- **Evidence:** Production code line 412: `saveAsTemplate: _saveAsTemplate` (no guard). Test at line 424: `bool effectiveSaveAsTemplate = saveAsTemplate && eventId.HasValue;` contradicts test at line 773: `bool effectiveSaveAsTemplate = saveAsTemplate;`.

### Finding 2: No test for null→DefaultSkillRating coercion at submit boundary

- **Severity:** Minor
- **Category:** correctness
- **Description:** TASK-002 changed `_skillRating` from `int` (default 50) to `int?` (null). The null-coalescing coercion `_skillRating ?? ValidationConstants.DefaultSkillRating` at line 147 of `CreatePlayerPage.razor.cs` is not directly tested. The existing `Submit_WithValidName_TrimsAndCallsService` test uses `It.IsAny<int>()` for the skill rating parameter, which doesn't verify the null→50 coercion. `Submit_PassesNicknameAndSkillRatingToService` tests an explicit value (75) but not the null default path.
- **Affected artifacts:** `tests/TourneyPal.Tests/Features/Players/CreatePlayer/CreatePlayerPageTests.cs`
- **Recommendation:** Add a test: `Submit_WithNullSkillRating_PassesDefaultSkillRating` that leaves the skill rating field unset and verifies the service receives `ValidationConstants.DefaultSkillRating` (50).
- **Evidence:** `CreatePlayerPage.razor.cs` line 147: `_skillRating ?? ValidationConstants.DefaultSkillRating`. No test in `CreatePlayerPageTests.cs` verifies this coercion path.

### Finding 3: Professional theme dark palette Secondary mismatches preview swatch

- **Severity:** Minor
- **Category:** correctness
- **Description:** The Professional theme's `PaletteDark.Secondary` is `#ff4081` (Material Design Pink A200), while the corresponding preview swatch `DarkColor` is `#FF375F` (closer to Apple dark-mode pink). TASK-007 updated preview swatches to Apple HIG values but did not update the dark palette Secondary. All other themes have swatches that match their palette values exactly. The swatch renders a color that doesn't appear in the actual theme.
- **Affected artifacts:** `src/TourneyPal.UI/Common/Theme/ThemeRegistry.cs` — Professional theme PaletteDark.Secondary (line ~129) vs PreviewSwatches[1].DarkColor (line ~155)
- **Recommendation:** Either update `PaletteDark.Secondary` to `#FF375F` (Apple HIG dark pink), or update the swatch `DarkColor` to `#ff4081` to match the actual palette. The former is preferred to maintain Apple HIG alignment.
- **Evidence:** `PaletteDark.Secondary = "#ff4081"` at line ~129. `new ThemePreviewSwatch(LightColor: "#FF2D55", DarkColor: "#FF375F")` at line ~155. Other themes (e.g., cyber-future) have matching swatch/palette values.

### Finding 4: Logic-level tests assert local variables rather than production code

- **Severity:** Minor
- **Category:** correctness
- **Description:** Several tests in `CreateGamePageTemplateSelectionTests` (e.g., `ActionSheetCancel_DoesNotCallCancel`, `TemplatePickerDialogCancel_AlwaysShowsBlankForm`, `TemplateContext_PickerCancel_ShowsBlankForm`) declare and assert local variables that replicate what the production code _should_ do, but don't actually exercise any production code path. For example, `ActionSheetCancel_DoesNotCallCancel` asserts `cancelWasCalled = false` where `cancelWasCalled` is a local boolean that was never set to true — this proves nothing about `OnActionSheetCancel()`. This is an acknowledged technical limitation (bUnit + MudBlazor v8 incompatibility) but these tests provide spec-documentation value only, not regression protection.
- **Affected artifacts:** `tests/TourneyPal.Tests/Features/Games/CreateGame/CreateGamePageTemplateSelectionTests.cs` — lines 166–175, 178–192, 194–210, etc.
- **Recommendation:** Add a comment block at the top of the class explicitly stating which tests are "spec documentation" vs "behavior verification." Consider whether `FormFieldState` and `GameValidators` tests (which DO call production code) could be split into a separate class for clarity.
- **Evidence:** `ActionSheetCancel_DoesNotCallCancel` (line ~166): `bool cancelWasCalled = false;` is never mutated, then asserted `Assert.False(cancelWasCalled)`.

## Summary

The 8 bug fixes are functionally correct. The null→DefaultSkillRating coercion (TASK-002) uses the `??` operator in the right place. The `OpenTemplatePickerDialog` await within `OnInitializedAsync` (TASK-005) works correctly — no lifecycle issues because the dialog is rendered by MudBlazor's overlay system. No race condition exists between `_flowStep` assignment and `StateHasChanged()` (TASK-004) — both are synchronous. Apple HIG hex values for semantic colors (TASK-007/008) are correctly applied across all 6 non-HC themes. The main concern is a contradictory pair of tests that document opposite specifications for the `saveAsTemplate` EventId guard, which should be cleaned up to prevent future confusion.
