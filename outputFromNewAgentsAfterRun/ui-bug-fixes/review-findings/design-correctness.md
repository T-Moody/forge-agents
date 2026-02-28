# Adversarial Review: design — correctness

## Review Metadata

- **Review Focus:** correctness
- **Risk Level:** 🟡
- **Round:** 1
- **Run ID:** 2026-02-26T14:30:00Z

## Overall Verdict

**Verdict:** needs_revision
**Confidence:** High

## Findings

### Finding 1: BUG-006 dark Background #000000 directly contradicts spec AC-006-8

- **Severity:** Critical
- **Category:** correctness
- **Description:** The design proposes Professional dark mode `Background = "#000000"` and `BackgroundGray = "#000000"`. However, the spec acceptance criterion AC-006-8 explicitly states: _"dark mode background is not pure black."_ Additionally, the existing test `AllNonHcThemes_PaletteDark_BackgroundIsNotPureBlack` in `ThemeRegistryTests.cs` line 217 asserts `Assert.NotEqual("#000000", theme.Theme.PaletteDark.Background.Value)` for all non-HC themes. The design proposes #000000 as "Apple HIG dark bg," and while Apple does use pure black on OLED screens, the project has explicitly established a non-pure-black dark mode invariant in both its spec and its test suite. Implementing this design will produce a failing test that the spec itself says must pass, and will violate the coded invariant.
- **Affected artifacts:** `design-output.yaml` (BUG-006 Professional PaletteDark Background/BackgroundGray), `design.md` §4.6 PaletteDark table
- **Recommendation:** Use a near-black dark background that satisfies the existing invariant. The current value `#1a1a27` or a value closer to Apple's elevated dark like `#1C1C1E` for Background (with a darker but non-pure-black BackgroundGray like `#141414`) would align with both Apple aesthetics and the project's invariant. Alternatively, if the team intentionally wants pure black, update AC-006-8 wording and the test, but that requires explicit spec revision.
- **Evidence:**
  - Spec AC-006-8: _"ThemeRegistryTests still pass — all 22 core properties are non-null, background hierarchy is maintained, dark mode background is not pure black."_
  - Test assertion at [ThemeRegistryTests.cs](tests/TourneyPal.Tests/Common/Theme/ThemeRegistryTests.cs#L225-L227): `Assert.NotEqual("#000000", theme.Theme.PaletteDark.Background.Value, StringComparer.OrdinalIgnoreCase)`
  - Design proposal at `design-output.yaml` line ~490: `Background: { from: "#1a1a27", to: "#000000" }` and `BackgroundGray: { from: "#151521", to: "#000000" }`

---

### Finding 2: BUG-006 light Surface #FFFFFF violates test invariant and omits DrawerBackground update

- **Severity:** Major
- **Category:** correctness
- **Description:** The design proposes Professional light `Surface = "#FFFFFF"`. Two existing test assertions will fail:
  1. `AllNonHcThemes_PaletteLight_Has22CoreProperties` at [ThemeRegistryTests.cs](tests/TourneyPal.Tests/Common/Theme/ThemeRegistryTests.cs#L202): `AssertPaletteColorNotEqual("#FFFFFF", palette.Surface)` — Surface must NOT be pure white.
  2. The same test at [line 203-204](tests/TourneyPal.Tests/Common/Theme/ThemeRegistryTests.cs#L203-L204) asserts `DrawerBackground == Surface`. The design changes Surface from `#F2F2F6` to `#FFFFFF` but does **not** include a corresponding `DrawerBackground` update. The current `DrawerBackground = "#F2F2F6"` would no longer match `Surface = "#FFFFFF"`, breaking this assertion.

  While Apple HIG does use #FFFFFF as the elevated secondary background, the project's test suite encodes a design rule that Surface should not be pure white (likely to maintain visual distinction between elevated cards and the browser/system white chrome). The design's testing section estimates "5-10 test updates" but does not flag this as an invariant conflict.

- **Affected artifacts:** `design-output.yaml` (BUG-006 Professional PaletteLight Surface change), `design.md` §4.6 PaletteLight table
- **Recommendation:** Either (a) use a near-white Surface like `#FAFAFA` or `#F9F9FB` that satisfies the invariant while approaching Apple's light elevated backgrounds, or (b) explicitly note that the Surface != #FFFFFF invariant must be relaxed with justification. Also add `DrawerBackground` to the change list to match whatever Surface value is chosen — the design must not omit this cascading update.
- **Evidence:**
  - Test assertion at [ThemeRegistryTests.cs](tests/TourneyPal.Tests/Common/Theme/ThemeRegistryTests.cs#L202): `AssertPaletteColorNotEqual("#FFFFFF", palette.Surface)`
  - DrawerBackground assertion at [line 203-204](tests/TourneyPal.Tests/Common/Theme/ThemeRegistryTests.cs#L203-L204): `Assert.Equal(palette.Surface.Value, palette.DrawerBackground.Value)`
  - Current theme values at [ThemeRegistry.cs](src/TourneyPal.UI/Common/Theme/ThemeRegistry.cs#L76-L82): `Surface = "#F2F2F6"`, `DrawerBackground = "#F2F2F6"`
  - Design proposal omits DrawerBackground from the PaletteLight change list entirely

---

### Finding 3: BUG-006 dark BackgroundGray == Background breaks hierarchy

- **Severity:** Major
- **Category:** correctness
- **Description:** The design proposes both `Background = "#000000"` and `BackgroundGray = "#000000"` for Professional dark mode. Having these two values identical eliminates the background hierarchy that the palette system relies on. BackgroundGray is semantically meant to be a distinct (typically darker) layer behind the main background. When both are #000000, any UI that differentiates between these two semantic layers will lose visual distinction. The current values (`Background = "#1a1a27"`, `BackgroundGray = "#151521"`) maintain a clear hierarchy. The spec AC-006-8 requires "background hierarchy is maintained," which this change violates.
- **Affected artifacts:** `design-output.yaml` (BUG-006 Professional PaletteDark BackgroundGray), `design.md` §4.6 PaletteDark table
- **Recommendation:** If Background remains near-black (not pure black), use a value for BackgroundGray that is slightly darker than Background to maintain the hierarchy. For example, if Background = `#1C1C1E`, then BackgroundGray could be `#121214`.
- **Evidence:**
  - Spec AC-006-8: _"background hierarchy is maintained"_
  - Design proposal: both Background and BackgroundGray → `#000000`
  - Current values in [ThemeRegistry.cs](src/TourneyPal.UI/Common/Theme/ThemeRegistry.cs#L118-L119): `Background = "#1a1a27"`, `BackgroundGray = "#151521"` — a clear 2-level hierarchy

---

### Finding 4: BUG-004 OnActionSheetCancel does not clear \_actionSheetActions

- **Severity:** Minor
- **Category:** correctness
- **Description:** The reference pattern `ParticipantsTabContent.CloseAddActionSheet()` clears the actions list (`_addActionSheetActions = []`) alongside hiding the sheet. The BUG-004 fix for `OnActionSheetCancel()` sets `_actionSheetVisible = false` and transitions to `FlowStep.Form` but does not clear `_actionSheetActions`. While this won't cause a functional bug (actions are rebuilt each time `ShowDecisionSheet()` is called), it leaves stale data in memory and deviates from the established pattern. If the component were to re-render the action sheet without calling `ShowDecisionSheet()`, stale actions could appear.
- **Affected artifacts:** `design-output.yaml` (BUG-004 OnActionSheetCancel after code), `design.md` §4.2 Change 1
- **Recommendation:** Add `_actionSheetActions = [];` to the fix for consistency with the reference pattern.
- **Evidence:**
  - Reference pattern: `ParticipantsTabContent.CloseAddActionSheet()` sets `_addActionSheetActions = []`
  - BUG-004 proposed fix omits action list reset
  - Side-by-side comparison in `design-output.yaml` shows the pattern discrepancy

---

### Finding 5: BUG-005 MudNumericField missing explicit T parameter

- **Severity:** Minor
- **Category:** correctness
- **Description:** The design changes `MudNumericField T="int"` to `MudNumericField Value="_skillRating"` without an explicit `T` type parameter. While Blazor can infer `T=int?` from the `Value` binding and the explicit lambda `(int? v)` in `ValueChanged`, relying on type inference for generic components can be fragile across Blazor/MudBlazor versions. The reference pattern in `CreateRosterPlayerPage.razor` also omits `T`, so this is consistent, but an explicit `T="int?"` would be more robust and self-documenting.
- **Affected artifacts:** `design-output.yaml` (BUG-005 Razor changes), `design.md` §4.5 Razor before/after
- **Recommendation:** Add `T="int?"` to the `MudNumericField` for explicitness: `<MudNumericField T="int?" Value="_skillRating" ...>`. This matches the project's convention of specifying type parameters on generic components.
- **Evidence:**
  - Current code at [CreatePlayerPage.razor](src/TourneyPal.UI/Features/Players/CreatePlayer/CreatePlayerPage.razor#L30): `MudNumericField T="int"` — explicit T is used
  - Proposed code removes T entirely — inconsistent with the pattern of the same file

---

## Summary

The design is well-structured overall, with thorough analysis of the P0 blocker (BUG-004) and sound patterns for BUG-001/002/003/005/007/008. However, BUG-006 (theme redesign) has a Critical spec contradiction: the proposed dark mode Background #000000 directly violates AC-006-8 which states "dark mode background is not pure black," and will fail an existing test invariant. Additionally, the proposed light Surface #FFFFFF violates a separate test invariant (Surface != #FFFFFF) and the design omits the required cascading DrawerBackground update. These issues must be resolved before implementation to avoid test failures and spec non-compliance. BUG-004 and BUG-005 have minor pattern deviations that should be addressed for consistency.
