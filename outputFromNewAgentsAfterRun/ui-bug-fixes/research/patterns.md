# Research: Patterns

## Summary

Analyzed 6 pattern areas across the codebase for the 5 reported bugs. Found 15 findings covering swipe icon rendering (JS+CSS, 24px icon in 80px container), template creation flows (dialog-based, not dropdown), bottom sheet cancel behavior differences (inline vs navigate-away), optional field handling (`int?` vs `int` inconsistency), theme color pipeline (ThemeRegistry → MudThemeProvider), and 3 additional inconsistencies. Each finding includes the correct pattern with code examples and comparison to the broken behavior.

---

## Findings

### F-1: Swipe delete icon uses JS innerHTML with MudBlazor mud-icon-size-medium (24x24px) in 80px container

**Category:** pattern

The swipe-to-delete indicator is rendered in `showDeleteIndicator()` at line 155 of `list-swipe-bridge.js`:

```javascript
// list-swipe-bridge.js line 160
indicator.innerHTML =
  '<span class="mud-icon-root"><svg viewBox="0 0 24 24" class="mud-svg-icon mud-icon-size-medium"><path d="M6 19c0 1.1.9 2 2 2h8c1.1 0 2-.9 2-2V7H6v12zM19 4h-3.5l-1-1h-5l-1 1H5v2h14V4z"/></svg></span>';
```

CSS container:

```css
/* list-swipe.css lines 68-82 */
.swipe-delete-indicator {
  width: 80px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.swipe-delete-indicator .mud-svg-icon {
  fill: var(--mud-palette-error-contrasttext);
  /* NO width/height override — defaults to MudBlazor's 24×24px */
}
```

MudBlazor icon size classes:

- `mud-icon-size-small` = 20×20px
- `mud-icon-size-medium` = 24×24px **(current — too small)**
- `mud-icon-size-large` = 35×35px

**Evidence:**

- `src/Tornimate.Mobile/wwwroot/js/list-swipe-bridge.js` lines 155-170
- `src/TourneyPal.UI/wwwroot/css/list-swipe.css` lines 68-93

**Relevance:** Fix should change CSS class to `mud-icon-size-large` in JS, or add explicit CSS width/height override. Single change propagates to all 5 list pages.

---

### F-2: MudBlazor icons in Razor components consistently use Size enum — JS bypass is the anomaly

**Category:** pattern

Throughout the codebase, MudBlazor icons use the `Size` parameter:

```razor
<MudIcon ... Size="Size.Large" />   <!-- GameCard, list rows -->
<MudIcon ... Size="Size.Small" />   <!-- TemplatePickerDialog -->
<MudIcon ... Size="Size.Medium" />  <!-- Icon buttons -->
```

The swipe delete indicator is the ONLY place where an icon is rendered via raw JS innerHTML, because it's created dynamically during touch events (outside Blazor rendering). CSS is the correct approach for sizing this icon.

**Evidence:**

- `src/TourneyPal.UI/Features/Events/GamesTabContent.razor` line 80 (Size.Large)
- `src/TourneyPal.UI/Features/Games/CreateGame/TemplatePickerDialog.razor` line 31 (Size.Small)
- JS innerHTML is the only non-component icon rendering path

**Relevance:** The fix should use CSS override for maintainability, matching the pattern of CSS-controlled indicator styling.

---

### F-3: Template selection uses TemplatePickerDialog (MudDialog) — not MudSelect or MudAutocomplete

**Category:** pattern

The app uses a full-screen `MudDialog` for template selection:

```csharp
// CreateGamePage.razor.cs — OpenTemplatePickerDialog()
var parameters = new DialogParameters<TemplatePickerDialog>
{
    { x => x.BuiltInTemplates, _builtInTemplates },
    { x => x.CustomTemplates, _customTemplates }
};
IDialogReference dialog = await DialogService.ShowAsync<TemplatePickerDialog>(
    Loc["Games_ChooseTemplateTitle"], parameters, options);
DialogResult? result = await dialog.Result;
```

The `TemplatePickerDialog.razor` contains:

- Search/filter text field
- Two grouped sections: "System Templates" and "My Templates"
- Each template is a full-width `MudButton` (min-height 44px)
- A "Create Custom Game" button at the bottom

No other "template/preset" selection flows exist in the app.

**Evidence:**

- `src/TourneyPal.UI/Features/Games/CreateGame/TemplatePickerDialog.razor` (full dialog)
- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor.cs` lines 230-270

**Relevance:** The existing pattern is a dialog-based picker, not a dropdown. Bug 2's "missing dropdown" may be a misunderstanding or a feature request for template cloning in settings context.

---

### F-4: Two-step flow pattern: ActionSheet → Form, with smart skip logic (DD-5r)

**Category:** pattern

The `CreateGamePage` has a two-step flow for event-scoped game creation:

```csharp
// OnInitializedAsync — event context:
if (_customTemplates.Any())
{
    _flowStep = FlowStep.Form;
    await OpenTemplatePickerDialog();  // Skip to dialog immediately
}
else
{
    _flowStep = FlowStep.ActionSheet;
    ShowDecisionSheet();  // Show "Use Template" / "Create Custom"
}

// Template context (settings):
else
{
    _flowStep = FlowStep.Form;  // Skip to blank form directly
}
```

Template management in settings goes straight to the blank form — this is intentional. The "missing dropdown" in settings means no pre-selection mechanism is offered. If cloning from existing templates is desired, it would need a new feature.

**Evidence:**

- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor.cs` lines 118-155
- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor.cs` lines 18-23 (FlowStep enum)

**Relevance:** Template creation in settings is intentionally a blank form. Adding a dropdown would be a feature enhancement, not a bug fix.

---

### F-5: Working bottom sheet pattern (ParticipantsTabContent) vs broken games tab pattern

**Category:** pattern

**WORKING PATTERN — ParticipantsTabContent:**

```csharp
// Shows action sheet on the SAME PAGE (tab content)
private void ShowAddActionSheet()
{
    _addActionSheetActions = [
        new() { Label = Loc["Players_AddPlayer"], Icon = ...,
            OnClick = () => { NavigateToAdd(); return Task.CompletedTask; } },
        new() { Label = Loc["Participants_ImportFromRoster"], ... },
        new() { Label = Loc["Players_AddMultiple"], ... }
    ];
    _addActionSheetVisible = true;
    StateHasChanged();
}

// Cancel just closes the sheet — no navigation
private void CloseAddActionSheet()
{
    _addActionSheetVisible = false;
    _addActionSheetActions = [];
    StateHasChanged();
}
```

```razor
<!-- Two separate ListActionSheet instances -->
<ListActionSheet IsVisible="_actionSheetVisible" ... OnCancel="CloseActionSheet" />
<ListActionSheet IsVisible="_addActionSheetVisible" ... OnCancel="CloseAddActionSheet" />
```

**DIFFERENT PATTERN — GamesTabContent:**

```csharp
// Navigates DIRECTLY to create page (no action sheet on tab)
private void NavigateToAddGame()
{
    NavigationManager.NavigateTo($"/events/{EventId}/games/add");
}
```

**THE BUG IS IN CreateGamePage (navigation target):**

```csharp
// CreateGamePage — cancel navigates AWAY
private void OnActionSheetCancel()
{
    _actionSheetVisible = false;
    Cancel();  // ← Navigates to /events/{EventId} — exits entire create flow
}
```

The participants tab avoids this by keeping the action sheet on the same page. The games tab navigates to a separate page first, then the page shows an action sheet whose cancel exits the entire page.

**Evidence:**

- `src/TourneyPal.UI/Features/Events/ParticipantsTabContent.razor.cs` lines 370-401
- `src/TourneyPal.UI/Features/Events/GamesTabContent.razor.cs` lines 353-356
- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor.cs` lines 218-222

**Relevance:** The 'black slide' and 'navigate back' bugs are in CreateGamePage's cancel handling. Fix options: (a) change cancel to show blank form instead of navigating away, or (b) restructure the games tab to use an inline action sheet like participants.

---

### F-6: ListActionSheet component pattern: HandleCancel closes sheet first, HandleAction closes then executes

**Category:** pattern

```csharp
// ListActionSheet.razor.cs
private async Task HandleCancel()
{
    await OnCancel.InvokeAsync();
}

private async Task HandleAction(ListAction action)
{
    await OnCancel.InvokeAsync(); // Close sheet first
    await Task.Yield();           // Allow render cycle to remove DOM
    await action.OnClick();       // Then execute action
}
```

The component itself only manages visibility semantics. The parent decides what cancel means:

- **ParticipantsTabContent:** Cancel = close sheet, stay on page ✓
- **CreateGamePage:** Cancel = close sheet AND navigate away ✗ (unexpected UX)

**Evidence:**

- `src/TourneyPal.UI/Common/Components/ListActionSheet.razor.cs` lines 48-57
- `docs/standards/ui-ux.md` section 8

**Relevance:** CreateGamePage's cancel should not navigate away when canceling the initial decision sheet.

---

### F-7: Roster add page uses nullable int? for optional skill rating — correct pattern

**Category:** pattern

**CreateRosterPlayerPage (correct optional pattern):**

```csharp
// Code-behind:
private int? _skillRating;  // nullable = truly optional

private void OnSkillRatingChanged(int? value)
{
    _skillRating = value;
    _fieldState.MarkTouched(nameof(_skillRating));
}

private string? ValidateSkillRating(int? value)
{
    if (!_fieldState.IsTouched(nameof(_skillRating))) return null;
    return RosterValidators.ValidateSkillRating(value, Loc);
}
```

```razor
<MudNumericField Value="_skillRating"
                 ValueChanged="@((int? v) => OnSkillRatingChanged(v))"
                 Label="@Loc["Players_SkillRating"]"
                 Validation="@(new Func<int?, string?>(ValidateSkillRating))"
                 Min="@ValidationConstants.RosterSkillRatingMin"
                 Max="@ValidationConstants.RosterSkillRatingMax"
                 Class="mb-4" Variant="Variant.Outlined"
                 HelperText="@Loc["Players_SkillRatingHelper"]" />
```

Validation accepts null as valid:

```csharp
public static string? ValidateSkillRating(int? value, IStringLocalizer<SharedResources>? loc)
{
    if (!value.HasValue) return null;  // Optional field — null is valid
    if (value < MinSkillRating) return "...";
    if (value > MaxSkillRating) return "...";
    return null;
}
```

Constants: `RosterSkillRatingMin = 1`, `RosterSkillRatingMax = 100`, `DefaultSkillRating = 50`.

**Evidence:**

- `src/TourneyPal.UI/Features/Roster/CreateRosterPlayer/CreateRosterPlayerPage.razor.cs` line 45
- `src/TourneyPal.UI/Features/Roster/CreateRosterPlayer/CreateRosterPlayerPage.razor` lines 38-41
- `src/TourneyPal.UI/Common/Validation/RosterValidators.cs` lines 115-131

**Relevance:** This is the canonical pattern for optional numeric fields. The CreatePlayerPage deviates from it.

---

### F-8: CreatePlayerPage uses non-nullable int for skill rating — inconsistency with roster pattern

**Category:** pattern

**CreatePlayerPage (event context — INCONSISTENT):**

```csharp
// Code-behind:
private int _skillRating = ValidationConstants.DefaultSkillRating;  // NON-nullable, defaults to 50
```

```razor
<MudNumericField T="int"
                 @bind-Value="_skillRating"
                 Label="@Loc["Player_SkillRating"]"
                 Min="@ValidationConstants.RosterSkillRatingMin"
                 Max="@ValidationConstants.RosterSkillRatingMax"
                 Class="mb-4" Variant="Variant.Outlined" />
```

**Issues:**

1. Uses `T="int"` (non-nullable) vs `int?` in roster page
2. Defaults to 50 — user cannot clear the field to "no rating"
3. No validation function — relies only on Min/Max
4. No `HelperText` explaining the field is optional
5. No `FormFieldState` deferred validation — uses simple `@bind-Value`

**Evidence:**

- `src/TourneyPal.UI/Features/Players/CreatePlayer/CreatePlayerPage.razor.cs` line 58
- `src/TourneyPal.UI/Features/Players/CreatePlayer/CreatePlayerPage.razor` lines 30-34
- Compare with `src/TourneyPal.UI/Features/Roster/CreateRosterPlayer/CreateRosterPlayerPage.razor.cs` line 45

**Relevance:** Fix should align CreatePlayerPage with the roster pattern: use `int?`, add validation, add HelperText, use Value/ValueChanged pattern with FormFieldState.

---

### F-9: Theme color system: ThemeRegistry → ThemeDefinition → MudTheme → MudThemeProvider pipeline

**Category:** pattern

The theme system follows a clean pipeline:

1. **ThemeRegistry** (static, 708 lines): 8 themes, each built by a `Build*Theme()` method
2. Each returns `ThemeDefinition` with `MudTheme` (PaletteLight + PaletteDark) + `PreviewSwatches`
3. **MainLayout** binds `<MudThemeProvider Theme="@Theme" IsDarkMode="IsDarkMode" />`
4. All colors flow through MudBlazor CSS variables (`--mud-palette-*`)
5. Components use `Color.Primary`, `Color.Error`, etc. — never hardcoded hex

**22 core palette properties per theme:**
Primary, Secondary, Tertiary, Background, Surface, BackgroundGray, AppbarBackground, AppbarText, DrawerBackground, DrawerText, DrawerIcon, TextPrimary, TextSecondary, TextDisabled, ActionDefault, Info, Success, Warning, Error, LinesDefault, TableLines, Divider.

**Evidence:**

- `src/TourneyPal.UI/Common/Theme/ThemeRegistry.cs` lines 1-150
- `src/TourneyPal.UI/Common/Theme/ThemeDefinition.cs`
- `tests/TourneyPal.Tests/Common/Theme/ThemeRegistryTests.cs`

**Relevance:** Color fixes are purely ThemeRegistry hex value changes. No CSS overrides, no component changes needed.

---

### F-10: Professional theme palette vs Apple HIG system colors — specific divergences

**Category:** pattern

| Property   | Current          | Apple HIG      | Status        |
| ---------- | ---------------- | -------------- | ------------- |
| Primary    | #594AE2 (purple) | #007AFF (blue) | **DIVERGENT** |
| Secondary  | #ff4081 (pink)   | #FF2D55 (pink) | Close         |
| Tertiary   | #1ec8a5 (teal)   | #5AC8FA (teal) | **DIVERGENT** |
| Error      | #ff3f5f          | #FF3B30        | Close         |
| Success    | #3dcb6c          | #34C759        | Close         |
| Warning    | #ffb545          | #FF9500        | Close         |
| Info       | #4a86ff          | #007AFF        | Close         |
| Background | #FAFAFC          | #F2F2F7        | Close         |
| Surface    | #F2F2F6          | #F2F2F7        | Nearly match  |

The biggest divergences are **Primary** (purple vs Apple blue) and **Tertiary**.

**Evidence:**

- `src/TourneyPal.UI/Common/Theme/ThemeRegistry.cs` lines 69-100 (PaletteLight), lines 108-136 (PaletteDark)

**Relevance:** Design decision: fully adopt Apple HIG (changes visual identity) or bring existing colors closer while keeping unique branding.

---

### F-11: PreviewSwatches must be updated to match any palette color changes

**Category:** pattern

Each `ThemeDefinition` includes `PreviewSwatches`:

```csharp
PreviewSwatches =
[
    new ThemePreviewSwatch(LightColor: "#594AE2", DarkColor: "#7e6fff"),  // Primary
    new ThemePreviewSwatch(LightColor: "#ff4081", DarkColor: "#ff4081"),   // Secondary
    new ThemePreviewSwatch(LightColor: "#1ec8a5", DarkColor: "#1ec8a5"),   // Tertiary
    new ThemePreviewSwatch(LightColor: "#F2F2F6", DarkColor: "#1e1e2d"),   // Surface
]
```

Updating palette hex values without updating PreviewSwatches creates a mismatch.

**Evidence:**

- `src/TourneyPal.UI/Common/Theme/ThemeRegistry.cs` lines 139-145
- `tests/TourneyPal.Tests/Common/Theme/ThemeRegistryTests.cs` line 82

**Relevance:** Any color change must update both palette AND preview swatches.

---

### F-12: Inconsistency: CreatePlayerPage skill rating is non-nullable (int) while all other forms use nullable (int?)

**Category:** pattern

Three pages handle skill rating:

| Page                   | Type       | Optional?           | Consistent? |
| ---------------------- | ---------- | ------------------- | ----------- |
| CreateRosterPlayerPage | `int?`     | Yes                 | ✓           |
| UpdateRosterPlayerPage | `int?`     | Yes                 | ✓           |
| CreatePlayerPage       | `int = 50` | No (forced default) | ✗           |

The `ParticipantsTabContent` shows `⭐ @player.SkillRating` chip only when `player.SkillRating != DefaultSkillRating`, so default 50 won't show the chip — but the player still has a non-null rating value stored.

**Evidence:**

- `src/TourneyPal.UI/Features/Players/CreatePlayer/CreatePlayerPage.razor.cs` line 58
- `src/TourneyPal.UI/Features/Roster/CreateRosterPlayer/CreateRosterPlayerPage.razor.cs` line 45
- `src/TourneyPal.UI/Features/Roster/UpdateRosterPlayer/UpdateRosterPlayerPage.razor.cs` line 48

**Relevance:** Fix should align to `int?` pattern across all pages.

---

### F-13: Dead CSS rule .delete-icon in list-swipe.css is a maintenance hazard

**Category:** pattern

```css
/* list-swipe.css line 90-93 — DEAD CODE */
.swipe-delete-indicator .delete-icon {
  font-size: 1.25rem;
  margin-bottom: 4px;
}
```

The JS uses `.mud-svg-icon` class, never `.delete-icon`. This was noted in the UX consistency overhaul and left as "outside task scope."

**Evidence:**

- `src/TourneyPal.UI/wwwroot/css/list-swipe.css` lines 90-93
- `docs/feature/ux-consistency-overhaul/memory/implementer-06.mem.md`

**Relevance:** Clean up opportunity during Bug 1 fix. Low risk.

---

### F-14: Test patterns: Theme tests use Assert.All; form tests use logic-level testing

**Category:** pattern

**Theme testing (ThemeRegistryTests):**

- 15+ tests asserting all 8 themes have required properties
- `Assert.All(themes, ...)` for bulk assertions
- Specific PaletteLight property tests for professional theme
- Dark mode: `BackgroundIsNotPureBlack` assertion

**Form testing (CreateGamePageTemplateSelectionTests):**

- Logic-level tests (not full bUnit render) due to MudBlazor v8 limitations
- Tests flow logic independently of rendering

**No JS/CSS tests** — manual visual verification required.

**Evidence:**

- `tests/TourneyPal.Tests/Common/Theme/ThemeRegistryTests.cs` (313 lines)
- `tests/TourneyPal.Tests/Features/Games/CreateGame/CreateGamePageTemplateSelectionTests.cs` (644 lines)

**Relevance:** Theme hex changes need test updates. Form behavior changes need new logic-level tests.

---

### F-15: FormFieldState deferred validation pattern — used consistently across create/edit forms

**Category:** pattern

```csharp
// Standard pattern across all forms:
private readonly FormFieldState _fieldState = new();

// 1. Create deferred validator
private string? ValidateX(string? value) =>
    _fieldState.CreateDeferredValidator(nameof(_x), validator)(value);

// 2. Mark touched on change
private void OnXChanged(string value)
{
    _x = value;
    _fieldState.MarkTouched(nameof(_x));
}

// 3. On submit, mark all touched
_fieldState.MarkAllTouched(nameof(_displayName), ...);
```

For nullable numeric fields, deferred validation is done inline:

```csharp
private string? ValidateSkillRating(int? value)
{
    if (!_fieldState.IsTouched(nameof(_skillRating))) return null;
    return RosterValidators.ValidateSkillRating(value, Loc);
}
```

Used in: CreateRosterPlayerPage, UpdateRosterPlayerPage, CreateGamePage. NOT used for skill rating in CreatePlayerPage (one of the inconsistencies).

**Evidence:**

- `src/TourneyPal.UI/Common/Validation/FormFieldState.cs` (95 lines)
- `src/TourneyPal.UI/Features/Roster/CreateRosterPlayer/CreateRosterPlayerPage.razor.cs` lines 97-103
- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor.cs` lines 163-169

**Relevance:** Any new form fields should follow this pattern. CreatePlayerPage skill rating fix should add FormFieldState tracking.

---

## Source Files Examined

- `src/Tornimate.Mobile/wwwroot/js/list-swipe-bridge.js`
- `src/TourneyPal.UI/wwwroot/css/list-swipe.css`
- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor`
- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor.cs`
- `src/TourneyPal.UI/Features/Games/CreateGame/TemplatePickerDialog.razor`
- `src/TourneyPal.UI/Features/Games/CreateGame/TemplatePickerDialog.razor.cs`
- `src/TourneyPal.UI/Features/Events/GamesTabContent.razor`
- `src/TourneyPal.UI/Features/Events/GamesTabContent.razor.cs`
- `src/TourneyPal.UI/Features/Events/ParticipantsTabContent.razor`
- `src/TourneyPal.UI/Features/Events/ParticipantsTabContent.razor.cs`
- `src/TourneyPal.UI/Features/Roster/CreateRosterPlayer/CreateRosterPlayerPage.razor`
- `src/TourneyPal.UI/Features/Roster/CreateRosterPlayer/CreateRosterPlayerPage.razor.cs`
- `src/TourneyPal.UI/Features/Roster/UpdateRosterPlayer/UpdateRosterPlayerPage.razor.cs`
- `src/TourneyPal.UI/Features/Players/CreatePlayer/CreatePlayerPage.razor`
- `src/TourneyPal.UI/Features/Players/CreatePlayer/CreatePlayerPage.razor.cs`
- `src/TourneyPal.UI/Features/Games/GamesPage.razor`
- `src/TourneyPal.UI/Features/Games/GamesPage.razor.cs`
- `src/TourneyPal.UI/Common/Components/ListActionSheet.razor`
- `src/TourneyPal.UI/Common/Components/ListActionSheet.razor.cs`
- `src/TourneyPal.UI/Common/Theme/ThemeRegistry.cs`
- `src/TourneyPal.UI/Common/Theme/ThemeDefinition.cs`
- `src/TourneyPal.UI/Common/Theme/UiConstants.cs`
- `src/TourneyPal.UI/Common/Validation/RosterValidators.cs`
- `src/TourneyPal.UI/Common/Validation/FormFieldState.cs`
- `src/TourneyPal.Common/Common/ValidationConstants.cs`
- `tests/TourneyPal.Tests/Common/Theme/ThemeRegistryTests.cs`
- `tests/TourneyPal.Tests/Features/Games/CreateGame/CreateGamePageTemplateSelectionTests.cs`
- `docs/standards/ui-ux.md`

## Research Metadata

- **Confidence Level:** high
- **Coverage Estimate:** Comprehensive coverage of all 6 requested pattern areas. Every finding includes actual code snippets from source files with specific line references.
- **Gaps:** No bracket/scoring template patterns exist to compare against (game templates are the only template system). Apple HIG dark mode colors are approximated from public documentation — exact values would need verification from Apple's developer resources.
