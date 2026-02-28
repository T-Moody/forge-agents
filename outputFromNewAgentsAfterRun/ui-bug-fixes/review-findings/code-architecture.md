# Adversarial Review: code — claude-opus-4.6

## Review Metadata

- **Review Focus:** architecture
- **Risk Level:** 🟢
- **Round:** 1
- **Run ID:** 2026-02-26T14:30:00Z

## Overall Verdict

**Verdict:** approve
**Confidence:** High

## Findings

### Finding 1: Professional theme PreviewSwatches dark Secondary color does not match PaletteDark.Secondary

- **Severity:** Minor
- **Category:** maintainability
- **Description:** The Professional theme's `PaletteDark.Secondary` is `#ff4081` (unchanged from before), but `PreviewSwatches[1].DarkColor` was updated to `#FF375F`. The preview swatch is supposed to represent the actual theme palette, so this mismatch means the theme preview card shows a color the user won't actually see when they apply the theme in dark mode.
- **Affected artifacts:** [ThemeRegistry.cs](src/TourneyPal.UI/Common/Theme/ThemeRegistry.cs#L111) (PaletteDark.Secondary = `#ff4081`) vs [ThemeRegistry.cs](src/TourneyPal.UI/Common/Theme/ThemeRegistry.cs#L143) (PreviewSwatches DarkColor = `#FF375F`)
- **Recommendation:** Either update `PaletteDark.Secondary` to `#FF375F` to match the swatch, or update the swatch DarkColor to `#ff4081` to match the palette. The Apple HIG dark-mode equivalent of iOS system pink (`#FF2D55` light) is `#FF375F`, so the swatch value is likely the intended one and `PaletteDark.Secondary` should be updated to `#FF375F`.
- **Evidence:** `PaletteDark.Secondary = "#ff4081"` at line 111; `PreviewSwatches[1] = new ThemePreviewSwatch(LightColor: "#FF2D55", DarkColor: "#FF375F")` at line 143. The diff shows light Secondary was intentionally changed from `#ff4081` to `#FF2D55`, and the preview swatch dark was changed from `#ff4081` to `#FF375F`, but PaletteDark.Secondary was left at `#ff4081`.

### Finding 2: Inconsistent text-secondary styling approach across the codebase

- **Severity:** Minor
- **Category:** maintainability
- **Description:** The fix correctly identifies that `Color="Color.Secondary"` on `MudText` resolves to the theme's accent/brand color (e.g., `#FF2D55`), not the muted text color (`TextSecondary`). The 4 modified components now use `Style="color: var(--mud-palette-text-secondary)"`. However, 30+ other `.razor` files still use `Color="Color.Secondary"` on `MudText` for descriptive/label text. This creates two parallel approaches to the same intent (muted helper text), with no central CSS class or shared mechanism. Future developers may continue using the incorrect `Color.Secondary` pattern since it's the dominant pattern in the codebase.
- **Affected artifacts:** [EmptyState.razor](src/TourneyPal.UI/Common/Components/EmptyState.razor#L23), [LoadingState.razor](src/TourneyPal.UI/Common/Components/LoadingState.razor#L19), [ThemePreviewCard.razor](src/TourneyPal.UI/Features/Settings/ThemePreviewCard.razor#L8), [AppearancePage.razor](src/TourneyPal.UI/Features/Settings/AppearancePage.razor#L19) — plus 30+ files still using `Color.Secondary` on MudText.
- **Recommendation:** Consider adding a shared CSS utility class (e.g., `.text-muted` or `.text-secondary`) that applies `color: var(--mud-palette-text-secondary)` and using it via `Class="text-secondary"` instead of inline `Style` attributes. This would centralize the pattern and avoid scattered inline styles. This is out of scope for the current bug-fix batch but should be tracked as technical debt.
- **Evidence:** `grep_search` for `Color.Secondary` in `.razor` files returns 30+ matches. `grep_search` for `color: var(--mud-palette-text-secondary)` returns only 6 matches (4 from this changeset + 2 pre-existing). The TASK-003 implementation report documents this gap with a full audit list but does not establish a shared mechanism.

### Finding 3: CreatePlayerPage lacks unsaved-changes guardrail present in CreateRosterPlayerPage

- **Severity:** Minor
- **Category:** design
- **Description:** The `CreateRosterPlayerPage` (the stated reference pattern) implements `IDisposable`, has `_isDisposed`/`_isCommitting` flags, a `HasUnsavedChanges` property, `ClearUnsavedChanges()`, and `ResetFormState()` for unsaved-changes navigation guardrails. `CreatePlayerPage` has none of these. While this inconsistency is **pre-existing** and not introduced by TASK-002, the task explicitly aimed to align `CreatePlayerPage` with `CreateRosterPlayerPage` for skill rating handling. The skill rating changes are correctly aligned, but the broader pattern divergence remains.
- **Affected artifacts:** [CreatePlayerPage.razor.cs](src/TourneyPal.UI/Features/Players/CreatePlayer/CreatePlayerPage.razor.cs) (no IDisposable, no HasUnsavedChanges) vs [CreateRosterPlayerPage.razor.cs](src/TourneyPal.UI/Features/Roster/CreateRosterPlayer/CreateRosterPlayerPage.razor.cs#L13) (full guardrail pattern)
- **Recommendation:** Track as separate technical debt. The TASK-002 scope was limited to skill rating field alignment and is correctly implemented within that scope.
- **Evidence:** `CreateRosterPlayerPage` implements `IDisposable` (line 13), has `HasUnsavedChanges` (line 53), `ClearUnsavedChanges()` (line 78), `_isCommitting` (line 16). `CreatePlayerPage` has none of these constructs.

## Summary

The 8 bug fixes are architecturally sound. The CreateGamePage cancel-navigation fix correctly separates dialog/action-sheet dismiss (show blank form) from explicit cancel (navigate away), maintaining the TaskPage pattern. The CreatePlayerPage skill rating field now follows the FormFieldState deferred-validation pattern from CreateRosterPlayerPage. ThemeRegistry preserves its static immutable architecture with consistent Apple HIG semantic status colors across themes. Three minor findings: a PreviewSwatches/PaletteDark color mismatch in the Professional theme, an inconsistent text-secondary styling approach (inline Style vs Color.Secondary), and a pre-existing guardrail pattern gap in CreatePlayerPage. None block deployment.
