# UI Bug Fixes Batch — Technical Design

**Feature slug:** `ui-bug-fixes`
**Design date:** 2026-02-26
**Agent:** Designer
**Status:** DONE (Round 2 — revised BUG-006 palette values)

---

## 1. Title & Summary

Technical design for 8 UI bug fixes covering swipe-to-delete icon sizing, game template management, event game creation workflow (P0 blocker), skill rating field behavior, theme palette redesign for Apple HIG compliance, Color.Secondary audit, and dead CSS cleanup.

**Design goals:**

- Fix the P0 workflow blocker (BUG-004) that prevents adding games to events
- Align skill rating UX to the established nullable-optional pattern
- Bring theme palettes into Apple HIG compliance while preserving theme personality
- Clean up dead CSS and Color.Secondary misuse on shared components

---

## 2. Context & Inputs

### Sources Used

- `docs/feature/ui-bug-fixes/spec-output.yaml` — 8 requirements (BUG-001 through BUG-008)
- `docs/feature/ui-bug-fixes/feature.md` — Human-readable specification
- `docs/feature/ui-bug-fixes/research/architecture.yaml` — 13 findings on code structure
- `docs/feature/ui-bug-fixes/research/impact.yaml` — 15 findings on blast radius and severity
- `docs/feature/ui-bug-fixes/research/dependencies.yaml` — 11 findings on dependency chains
- `docs/feature/ui-bug-fixes/research/patterns.yaml` — 15 findings on codebase patterns
- `.github/instructions/blazor.instructions.md` — Component patterns
- `.github/instructions/components.instructions.md` — Card/Form/Dialog conventions
- `.github/instructions/csharp.instructions.md` — C# coding standards

### Key Source Files Examined

- `src/Tornimate.Mobile/wwwroot/js/list-swipe-bridge.js` (lines 145-180)
- `src/TourneyPal.UI/wwwroot/css/list-swipe.css` (lines 60-100)
- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor.cs` (full file, 478 lines)
- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor` (full file, 106 lines)
- `src/TourneyPal.UI/Features/Players/CreatePlayer/CreatePlayerPage.razor.cs` (full file, 171 lines)
- `src/TourneyPal.UI/Features/Players/CreatePlayer/CreatePlayerPage.razor` (full file, 57 lines)
- `src/TourneyPal.UI/Features/Roster/CreateRosterPlayer/CreateRosterPlayerPage.razor.cs` (lines 1-100)
- `src/TourneyPal.UI/Features/Roster/CreateRosterPlayer/CreateRosterPlayerPage.razor` (lines 30-45)
- `src/TourneyPal.UI/Common/Theme/ThemeRegistry.cs` (full file, 708 lines)
- `src/TourneyPal.UI/Features/Settings/ThemePreviewCard.razor` (full file)
- `src/TourneyPal.UI/Features/Settings/AppearancePage.razor` (full file)
- `src/TourneyPal.UI/Common/Components/EmptyState.razor` (full file)
- `src/TourneyPal.UI/Common/Components/LoadingState.razor` (full file)

---

## 3. High-Level Architecture

No architectural changes are needed. All fixes operate within existing component boundaries:

| Bug     | Layer affected            | Files modified               | Blast radius                            |
| ------- | ------------------------- | ---------------------------- | --------------------------------------- |
| BUG-001 | CSS only                  | 1 CSS file                   | 5 list pages (shared)                   |
| BUG-002 | UI code-behind            | 1 .razor.cs file             | Settings template creation              |
| BUG-003 | Investigation             | TBD                          | TBD                                     |
| BUG-004 | UI code-behind            | 1 .razor.cs file             | Event game creation                     |
| BUG-005 | UI code-behind + Razor    | 2 files                      | Event player creation                   |
| BUG-006 | Theme registry + 2 Razor  | 3 files                      | All themed UI (auto-propagate)          |
| BUG-007 | 2 shared Razor components | 2 files                      | All pages using EmptyState/LoadingState |
| BUG-008 | CSS only                  | 1 CSS file (same as BUG-001) | None (dead code removal)                |

---

## 4. Detailed Design Per Bug

### 4.1 BUG-001 + BUG-008: Swipe Delete Icon Sizing & Dead CSS Cleanup

**Decision D-1:** Use CSS `width/height` override on `.mud-svg-icon` (not JS class change).

**Root cause:** JS `innerHTML` in `list-swipe-bridge.js` line 161 creates an SVG icon with class `mud-icon-size-medium` (24×24px) inside an 80px container. After the UX overhaul removed the "Delete" text label, the icon appears undersized. A dead `.delete-icon` CSS rule exists but is never matched.

**Fix in `list-swipe.css`:**

```css
/* BEFORE (lines 88-94): */
.swipe-delete-indicator .mud-svg-icon {
  fill: var(--mud-palette-error-contrasttext);
}

.swipe-delete-indicator .delete-icon {
  font-size: 1.25rem;
  margin-bottom: 4px;
}

/* AFTER: */
.swipe-delete-indicator .mud-svg-icon {
  fill: var(--mud-palette-error-contrasttext);
  width: 36px;
  height: 36px;
}

/* .delete-icon rule REMOVED — dead code, JS uses .mud-svg-icon */
```

**Why 36px?** Matches MudBlazor's `mud-icon-size-large` (35px) while providing a round number that centers well in the 80px container (22px padding each side). Meets the ≥32px acceptance criterion.

**Propagation:** Single CSS change propagates to all 5 list pages: EventsPage, RosterPage, GamesPage, ParticipantsTabContent, GamesTabContent.

**Risk:** 🟢 — CSS-only, no .NET changes, zero test breakage.

---

### 4.2 BUG-004: P0 Bottom Sheet Navigate-Away Fix (PRIORITY)

**Decision D-4:** Decouple action sheet/dialog dismiss from page navigation.

**Root cause analysis — side-by-side comparison:**

#### Working Pattern: ParticipantsTabContent

```csharp
// src/TourneyPal.UI/Features/Events/ParticipantsTabContent.razor.cs
private void CloseAddActionSheet()
{
    _addActionSheetVisible = false;
    _addActionSheetActions = [];
    StateHasChanged();
}
// Result: Sheet closes. User stays on the page. ✅
```

#### Broken Pattern: CreateGamePage

```csharp
// src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor.cs (lines 219-222)
private void OnActionSheetCancel()
{
    _actionSheetVisible = false;
    Cancel();  // ← Cancel() navigates to /events/{EventId} ❌
}

// Cancel() at lines 356-366:
private void Cancel()
{
    if (!string.IsNullOrEmpty(TaskKey))
    {
        TaskResultService.SetResult(TaskKey, TaskResult.Canceled());
    }
    if (EventId.HasValue)
    {
        NavigationManager.NavigateTo($"/events/{EventId}");
    }
    else
    {
        NavigationManager.NavigateTo("/settings/game-templates");
    }
}
```

Also broken: `OpenTemplatePickerDialog()` cancel path (lines 249-258):

```csharp
if (result is null || result.Canceled)
{
    if (!_customTemplates.Any())
    {
        Cancel();       // ← Navigates away even from dialog cancel ❌
        return;
    }
    _flowStep = FlowStep.Form;
    _sourceTemplateName = null;
    StateHasChanged();
    return;
}
```

#### Fix:

**Change 1 — OnActionSheetCancel:**

```csharp
private void OnActionSheetCancel()
{
    _actionSheetVisible = false;
    _flowStep = FlowStep.Form;
    _sourceTemplateName = null;
    StateHasChanged();
}
```

**Change 2 — OpenTemplatePickerDialog cancel path:**

```csharp
if (result is null || result.Canceled)
{
    // Always show blank form on cancel — never navigate away
    _flowStep = FlowStep.Form;
    _sourceTemplateName = null;
    StateHasChanged();
    return;
}
```

**Cancel() remains unchanged** — it's still called by the TaskPage back/cancel button, which is the only legitimate exit from CreateGamePage.

**The user flow after fix:**

1. Navigate to `/events/{EventId}/games/add`
2. If no custom templates → ListActionSheet appears with overlay
3. User taps overlay or cancel → sheet closes, blank form appears ✅
4. User taps "Use Template" → TemplatePickerDialog opens
5. User cancels dialog → dialog closes, blank form appears ✅
6. User fills form → submits → game created
7. Only the toolbar back button navigates away

**Risk:** 🟡 — Modifies cancel semantics; test updates needed but pattern is proven.

---

### 4.3 BUG-002: Template Picker in Settings Context

**Decision D-2:** Reuse `TemplatePickerDialog` in template context.

**Change in `OnInitializedAsync()` (lines 148-162):**

```csharp
// BEFORE:
else
{
    // Template context: skip to form directly
    _flowStep = FlowStep.Form;
}

// AFTER:
else
{
    // Template context: offer template picker if templates exist
    if (_builtInTemplates.Any() || _customTemplates.Any())
    {
        _flowStep = FlowStep.Form;
        await OpenTemplatePickerDialog();
    }
    else
    {
        _flowStep = FlowStep.Form;
    }
}
```

**Dependency on BUG-004:** The `OpenTemplatePickerDialog()` cancel path must be fixed first (BUG-004 Change 2). Without BUG-004 fix, canceling the picker in template context would call `Cancel()` which navigates to `/settings/game-templates`.

**Edge case (EC-1):** If no templates exist at all (neither built-in nor custom), the picker is skipped and the user goes directly to the blank form.

**Risk:** 🟡 — Coupled with BUG-004 changes in same file.

---

### 4.4 BUG-003: Template Creation Error Investigation

**Decision D-3:** Integration-test-first diagnosis.

**Investigation plan:**

1. **Write integration test** exercising `CreateGameCommand` with `EventId = null`, valid name, `maxPlayersPerMatch = 4`:
   - File: `tests/TourneyPal.Tests/Features/Games/CreateGame/CreateGameHandlerTests.cs`
   - Test name: `Handle_TemplateCreation_WithNullEventId_Succeeds`
   - If test **fails** → backend root cause identified (handler, service, or DB constraint)
   - If test **passes** → bug is UI-side

2. **If UI-side**, inspect `CreateGamePage.Submit()`:
   - Check if `_form.Validate()` fails when form state is affected by BUG-004 flow
   - Check if `IsSubmitDisabled()` returns true when `_selectedTemplateValues` is null and `_isValid` is false (form never validated because action sheet was showing)
   - Current code path: action sheet shows → user dismisses → but `_form` may not be rendered yet → Submit fails

3. **Hypothesis:** The error may be a symptom of BUG-004. When the action sheet cancel navigates away, the user never reaches the form. If they somehow do reach it (e.g., via "Create Custom" action), the form state may be inconsistent because `_flowStep` transitions were interrupted by the cancel-navigation bug.

4. **If still unclear**, use Playwright automation on Android emulator to capture the exact error message.

**Risk:** 🟡 — Unknown root cause requires runtime investigation.

---

### 4.5 BUG-005: Optional Skill Rating

**Decision D-5:** Align to `CreateRosterPlayerPage` nullable `int?` pattern.

**Reference pattern from CreateRosterPlayerPage:**

```csharp
// Code-behind:
private int? _skillRating;  // nullable, starts empty

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

// Submit: passes _skillRating ?? DefaultSkillRating to service
```

```razor
<!-- Razor: -->
<MudNumericField Value="_skillRating"
                 ValueChanged="@((int? v) => OnSkillRatingChanged(v))"
                 Label="@Loc["Players_SkillRating"]"
                 Validation="@(new Func<int?, string?>(ValidateSkillRating))"
                 Min="@ValidationConstants.RosterSkillRatingMin"
                 Max="@ValidationConstants.RosterSkillRatingMax"
                 Class="mb-4" Variant="Variant.Outlined"
                 HelperText="@Loc["Players_SkillRatingHelper"]" />
```

**Changes to CreatePlayerPage.razor.cs:**

| Line         | Before                                                               | After                                                    |
| ------------ | -------------------------------------------------------------------- | -------------------------------------------------------- |
| 58           | `private int _skillRating = ValidationConstants.DefaultSkillRating;` | `private int? _skillRating;`                             |
| new          | —                                                                    | Add `OnSkillRatingChanged(int? value)` method            |
| new          | —                                                                    | Add `ValidateSkillRating(int? value)` method             |
| 130 (Submit) | `_skillRating` passed to CreatePlayerAsync                           | `_skillRating ?? ValidationConstants.DefaultSkillRating` |

**Changes to CreatePlayerPage.razor:**

```razor
<!-- BEFORE (lines 30-34): -->
<MudNumericField T="int"
                 @bind-Value="_skillRating"
                 Label="@Loc["Player_SkillRating"]"
                 Min="@ValidationConstants.RosterSkillRatingMin"
                 Max="@ValidationConstants.RosterSkillRatingMax"
                 Class="mb-4" Variant="Variant.Outlined" />

<!-- AFTER: -->
<MudNumericField T="int?"
                 Value="_skillRating"
                 ValueChanged="@((int? v) => OnSkillRatingChanged(v))"
                 Label="@Loc["Player_SkillRating"]"
                 Validation="@(new Func<int?, string?>(ValidateSkillRating))"
                 Min="@ValidationConstants.RosterSkillRatingMin"
                 Max="@ValidationConstants.RosterSkillRatingMax"
                 Class="mb-4" Variant="Variant.Outlined"
                 HelperText="@Loc["Players_SkillRatingHelper"]" />
```

**No changes to:** `CreatePlayerCommand`, `CreatePlayerValidator`, `CreatePlayerHandler`, `IPlayerDataService`, or `Player` entity. The null→default coercion happens at the UI submission boundary.

**Risk:** 🟢 — UI-only, proven pattern.

---

### 4.6 BUG-006: Theme Palette Redesign (REVISED — Round 2)

**Decision D-6:** Standardize semantic colors to Apple HIG, retain per-theme Primary identity, use off-black/off-white for structural colors.

> **Round 2 revision:** Addresses Critical/Major findings from adversarial review:
>
> - Dark Background `#000000` → `#1C1C1E` (satisfies AC-006-8 + `AllNonHcThemes_PaletteDark_BackgroundIsNotPureBlack`)
> - Dark BackgroundGray `#000000` → `#141416` (maintains hierarchy, not pure black)
> - Dark Surface `#1C1C1E` → `#2C2C2E` (shifted up since Background took `#1C1C1E`)
> - Light Surface `#FFFFFF` → `#FAFAFE` (satisfies `Surface != #FFFFFF` test invariant)
> - DrawerBackground explicitly cascaded in both light and dark palettes
> - PreviewSwatches updated to match revised Surface values

#### Apple HIG Reference Colors

| Color                | Light   | Dark    |
| -------------------- | ------- | ------- |
| System Blue          | #007AFF | #0A84FF |
| System Green         | #34C759 | #30D158 |
| System Red           | #FF3B30 | #FF453A |
| System Orange        | #FF9500 | #FF9F0A |
| System Teal          | #5AC8FA | #64D2FF |
| System Pink          | #FF2D55 | #FF375F |
| Primary Background   | #F2F2F7 | #000000 |
| Secondary Background | #FFFFFF | #1C1C1E |
| Tertiary Background  | —       | #2C2C2E |

> **Design adaptation:** Apple HIG uses `#000000` (dark bg) and `#FFFFFF` (light surface) which violate existing test invariants (`Background != #000000`, `Surface != #FFFFFF`). The design maps to Apple's secondary/tertiary layers instead, achieving the same layered aesthetic without breaking quality gates.

#### Professional Theme — Proposed Changes

**PaletteLight:**

| Property             | Current                | Proposed                    | Rationale                                                                                                       |
| -------------------- | ---------------------- | --------------------------- | --------------------------------------------------------------------------------------------------------------- |
| **Primary**          | `#594AE2` (purple)     | `#3478F6` (blue)            | Shift toward Apple system blue while retaining a slightly richer blue. Not pure #007AFF to maintain uniqueness. |
| **Secondary**        | `#ff4081` (pink)       | `#FF2D55` (Apple pink)      | Align to Apple system pink. Very close to current.                                                              |
| **Tertiary**         | `#1ec8a5` (teal-green) | `#5AC8FA` (Apple teal)      | Align to Apple system teal.                                                                                     |
| **Background**       | `#FAFAFC`              | `#F2F2F7` (Apple bg)        | Apple primary background.                                                                                       |
| **Surface**          | `#F2F2F6`              | `#FAFAFE` (off-white)       | Near-white surface. Satisfies `Surface != #FFFFFF` test invariant.                                              |
| **DrawerBackground** | `#F2F2F6`              | `#FAFAFE`                   | Matches Surface (existing light-mode convention enforced by tests).                                             |
| **BackgroundGray**   | `#E8E8EC`              | `#E5E5EA` (Apple separator) | Slightly warmer gray.                                                                                           |
| **Error**            | `#ff3f5f`              | `#FF3B30` (Apple red)       | Align to Apple system red.                                                                                      |
| **Success**          | `#3dcb6c`              | `#34C759` (Apple green)     | Align to Apple system green.                                                                                    |
| **Warning**          | `#ffb545`              | `#FF9500` (Apple orange)    | Align to Apple system orange.                                                                                   |
| **Info**             | `#4a86ff`              | `#007AFF` (Apple blue)      | Align to Apple system blue.                                                                                     |

**Light mode invariant compliance:**

- `Surface (#FAFAFE) != #FFFFFF` ✅
- `DrawerBackground (#FAFAFE) == Surface (#FAFAFE)` ✅ (light convention)
- `Background (#F2F2F7) != Surface (#FAFAFE)` ✅ (hierarchy maintained)

**PaletteDark:**

| Property             | Current              | Proposed                         | Rationale                                                               |
| -------------------- | -------------------- | -------------------------------- | ----------------------------------------------------------------------- |
| **Primary**          | `#7e6fff`            | `#5E9CFF`                        | Accessible blue for dark mode.                                          |
| **Tertiary**         | `#1ec8a5`            | `#64D2FF` (Apple teal dark)      | Align to Apple dark teal.                                               |
| **Background**       | `#1a1a27`            | `#1C1C1E` (Apple secondary dark) | Apple's secondary dark bg. Satisfies `Background != #000000` invariant. |
| **Surface**          | `#1e1e2d`            | `#2C2C2E` (Apple tertiary dark)  | Apple's tertiary dark bg. Maintains `Background < Surface` hierarchy.   |
| **BackgroundGray**   | `#151521`            | `#141416`                        | Darker than Background, maintains 3-level hierarchy. Not pure black.    |
| **DrawerBackground** | `#1a1a27`            | `#1C1C1E`                        | Matches Background (existing dark-mode convention).                     |
| **AppbarBackground** | `rgba(26,26,39,0.8)` | `rgba(28,28,30,0.8)`             | Updated RGB to match new Background (#1C1C1E = rgb(28,28,30)).          |
| **Error**            | `#ff3f5f`            | `#FF453A` (Apple red dark)       | Align to Apple dark red.                                                |
| **Success**          | `#3dcb6c`            | `#30D158` (Apple green dark)     | Align to Apple dark green.                                              |
| **Warning**          | `#ffb545`            | `#FF9F0A` (Apple orange dark)    | Align to Apple dark orange.                                             |
| **Info**             | `#4a86ff`            | `#0A84FF` (Apple blue dark)      | Align to Apple dark blue.                                               |

**Dark mode invariant compliance:**

- `Background (#1C1C1E) != #000000` ✅
- `BackgroundGray (#141416) < Background (#1C1C1E) < Surface (#2C2C2E)` ✅ (proper hierarchy)
- `DrawerBackground (#1C1C1E) == Background (#1C1C1E)` ✅ (dark convention)

**PreviewSwatches update:**

```csharp
// BEFORE:
new ThemePreviewSwatch(LightColor: "#594AE2", DarkColor: "#7e6fff"),  // Primary
new ThemePreviewSwatch(LightColor: "#ff4081", DarkColor: "#ff4081"),   // Secondary
new ThemePreviewSwatch(LightColor: "#1ec8a5", DarkColor: "#1ec8a5"),   // Tertiary
new ThemePreviewSwatch(LightColor: "#F2F2F6", DarkColor: "#1e1e2d"),   // Surface

// AFTER:
new ThemePreviewSwatch(LightColor: "#3478F6", DarkColor: "#5E9CFF"),   // Primary
new ThemePreviewSwatch(LightColor: "#FF2D55", DarkColor: "#FF375F"),   // Secondary
new ThemePreviewSwatch(LightColor: "#5AC8FA", DarkColor: "#64D2FF"),   // Tertiary
new ThemePreviewSwatch(LightColor: "#FAFAFE", DarkColor: "#2C2C2E"),   // Surface
```

#### Other Themes — Semantic Color Standardization

All 6 non-HC themes receive standardized Apple HIG semantic colors:

| Theme          | Error Light→        | Success Light→      | Warning Light→      | Info Light→         |
| -------------- | ------------------- | ------------------- | ------------------- | ------------------- |
| cyber-future   | `#FF1744`→`#FF3B30` | `#00C853`→`#34C759` | `#FF9100`→`#FF9500` | `#2196F3`→`#007AFF` |
| classic-sports | `#D32F2F`→`#FF3B30` | `#2E7D32`→`#34C759` | `#F57F17`→`#FF9500` | `#1976D2`→`#007AFF` |
| stadium-night  | `#C0392B`→`#FF3B30` | `#27AE60`→`#34C759` | `#D4A017`→`#FF9500` | `#2980B9`→`#007AFF` |
| retro-arcade   | `#FF1744`→`#FF3B30` | `#00C853`→`#34C759` | `#FFD600`→`#FF9500` | `#2979FF`→`#007AFF` |
| minimal-soft   | `#EF9A9A`→`#FF3B30` | `#A5D6A7`→`#34C759` | `#FFE082`→`#FF9500` | `#90CAF9`→`#007AFF` |
| nature-fresh   | `#E53935`→`#FF3B30` | `#43A047`→`#34C759` | `#F9A825`→`#FF9500` | `#039BE5`→`#007AFF` |

Dark mode equivalents: Error→`#FF453A`, Success→`#30D158`, Warning→`#FF9F0A`, Info→`#0A84FF`.

**High-contrast theme is NOT modified** — it follows WCAG AAA guidelines with its own distinct color system.

#### Color.Secondary on MudText Fix (ThemePreviewCard + AppearancePage)

```razor
<!-- ThemePreviewCard.razor line 9 -->
<!-- BEFORE: -->
<MudText Typo="Typo.body2" Color="Color.Secondary">@Loc[Theme.DescriptionKey]</MudText>
<!-- AFTER: -->
<MudText Typo="Typo.body2" Style="color: var(--mud-palette-text-secondary)">@Loc[Theme.DescriptionKey]</MudText>

<!-- AppearancePage.razor line 19 -->
<!-- BEFORE: -->
<MudText Typo="Typo.body2" Color="Color.Secondary">
<!-- AFTER: -->
<MudText Typo="Typo.body2" Style="color: var(--mud-palette-text-secondary)">
```

**Risk:** 🟡 — Large scope (708-line file, 8 themes). Theme-by-theme implementation with test verification.

---

### 4.7 BUG-007: Color.Secondary Audit

**Decision D-7:** Fix high-visibility shared components; document remainder.

**Files fixed in this batch (6 instances):**

| File                   | Line | Element             | Fix                                                                            |
| ---------------------- | ---- | ------------------- | ------------------------------------------------------------------------------ |
| EmptyState.razor       | 23   | Title MudText       | `Color="Color.Secondary"` → `Style="color: var(--mud-palette-text-secondary)"` |
| EmptyState.razor       | 28   | Description MudText | Same fix                                                                       |
| LoadingState.razor     | 19   | Circular text       | Same fix                                                                       |
| LoadingState.razor     | 29   | Linear text         | Same fix                                                                       |
| ThemePreviewCard.razor | 9    | Description text    | Fixed via BUG-006                                                              |
| AppearancePage.razor   | 19   | Mode label text     | Fixed via BUG-006                                                              |

**Full audit — Color.Secondary on MudText (muted text intent, NOT fixed in this batch):**

| File                                   | Instances | Notes                                                      |
| -------------------------------------- | --------- | ---------------------------------------------------------- |
| TaskPage.razor                         | 1         | Loading text                                               |
| ListActionSheet.razor                  | 1         | Title text                                                 |
| SettingsPage.razor                     | ~10       | Description, version, licence details                      |
| BracketSetupPage.razor                 | ~10       | Algorithm descriptions, captions                           |
| MatchPage.razor                        | 1         | Game description (also 1 MudButton — correct accent usage) |
| GameSetupStepsComponent.razor          | 4         | Step descriptions                                          |
| GameSetupSummaryComponent.razor        | 4         | Summary labels                                             |
| GamesTabContent.razor                  | 2         | Overline + description                                     |
| OverviewTabContent.razor               | 1         | Status caption (also 1 MudIcon — correct accent)           |
| GameDetailsPage.razor                  | 2         | Description text                                           |
| StandingsTable.razor                   | 3         | Empty state + detail text                                  |
| ExportPage.razor                       | 2         | Description + caption                                      |
| ParticipantsTabContent.razor           | 1         | Nickname text                                              |
| RosterPage.razor                       | 1         | Nickname text                                              |
| GamesPage.razor                        | 1         | Description text                                           |
| IconPickerDialog.razor                 | 3         | Category label, empty state, icon name                     |
| And ~15 more feature-specific files... |           |                                                            |

**Preserved (accent intent, NOT bugs):**

| File                           | Element                     | Why keep                      |
| ------------------------------ | --------------------------- | ----------------------------- |
| GameCard.razor                 | MudChip `Color.Secondary`   | Accent-colored chip (correct) |
| TiebreakerEditor.razor         | MudChip `Color.Secondary`   | Accent-colored chip (correct) |
| MatchPage.razor                | MudButton `Color.Secondary` | Accent button (correct)       |
| PlayerSkillRatingEditor.razor  | MudButton `Color.Secondary` | Text button (correct)         |
| StatDefinitionList.razor       | MudButton `Color.Secondary` | Icon button (correct)         |
| OverviewTabContent.razor       | MudIcon `Color.Secondary`   | Accent icon (correct)         |
| PlayerExclusionManager.razor   | MudIcon `Color.Secondary`   | Swap icon (correct)           |
| GenerateTeamsPage/Dialog.razor | MudSwitch `Color.Secondary` | Toggle accent (correct)       |

**Risk:** 🟢 — No tests assert on Color; shared components propagate fix to all pages.

---

## 5. Sequence / Interaction Notes

### BUG-004 + BUG-002 Interaction in CreateGamePage

The `OnInitializedAsync` method handles both event context and template context:

```
OnInitializedAsync()
├── Load builtInTemplates + customTemplates (shared by both contexts)
├── if EventId.HasValue (event context)
│   ├── Load event dates
│   ├── if customTemplates exist → OpenTemplatePickerDialog() → Form
│   └── else → ShowDecisionSheet() → ActionSheet
├── else (template context — BEFORE BUG-002 FIX)
│   └── _flowStep = Form (blank form, no picker)
└── else (template context — AFTER BUG-002 FIX)
    ├── if templates exist → OpenTemplatePickerDialog() → Form
    └── else → _flowStep = Form (blank form)
```

**Critical ordering:** BUG-004 must change `OnActionSheetCancel` and `OpenTemplatePickerDialog` cancel before BUG-002 adds the template picker invocation to the template context branch. Otherwise the cancel from the newly added picker would still navigate away.

---

## 6. Security Considerations

No security implications. All changes are:

- CSS/visual styling (BUG-001, BUG-008)
- UI navigation behavior (BUG-002, BUG-004)
- Form field type change (BUG-005)
- Static color hex values (BUG-006)
- Component attribute changes (BUG-007)

No authentication, authorization, data access, or API changes.

---

## 7. Failure & Recovery

| Failure Mode                                | Likelihood | Impact | Recovery                                                                 |
| ------------------------------------------- | ---------- | ------ | ------------------------------------------------------------------------ |
| BUG-004 fix breaks template selection flow  | Low        | High   | Tests cover both cancel and selection paths. Revert 2 methods if needed. |
| BUG-006 theme palette breaks visual harmony | Medium     | Medium | Theme-by-theme implementation with visual verification.                  |
| BUG-003 root cause is unexpectedly complex  | Medium     | Low    | Defer to Playwright investigation. Not on critical tournament path.      |
| BUG-005 null coercion missed in Submit path | Low        | Low    | Validator catches out-of-range. Default parameter provides fallback.     |
| BUG-002 template picker shows empty state   | Low        | Medium | Guard: skip picker when no templates exist (see code).                   |

---

## 8. Non-Functional Requirements

- **Performance:** No regressions. CSS changes are static. Theme values are compile-time constants. Form field type change is negligible.
- **Offline:** All fixes work offline. No network dependencies introduced.
- **Accessibility:** Swipe icon at 36px meets 44×44px container touch target. Theme colors maintain WCAG AA contrast ratios. HelperText on skill rating field improves discoverability.
- **Localization:** Skill rating HelperText uses existing `Players_SkillRatingHelper` localization key (already present for roster page). No new localization keys needed.
- **Backward compatibility:** No data schema changes. Existing events, games, players, and templates unaffected.

---

## 9. Migration & Backwards Compatibility

No migrations needed. No database changes. No API changes.

The theme palette changes affect visual appearance only. Users who selected a specific theme will see updated colors on next app launch. No action required from users.

---

## 10. Testing Strategy

### Test Updates by Bug

| Bug         | Test File                               | Updates Needed                                                                                                                                                           | New Tests Needed       |
| ----------- | --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------- |
| BUG-001/008 | None                                    | 0 (CSS only)                                                                                                                                                             | 0                      |
| BUG-002     | CreateGamePageTemplateSelectionTests.cs | 2-3 tests                                                                                                                                                                | 1-2 tests              |
| BUG-003     | CreateGameHandlerTests.cs               | 0                                                                                                                                                                        | 1 (investigation test) |
| BUG-004     | CreateGamePageTemplateSelectionTests.cs | 2-3 tests                                                                                                                                                                | 2-3 tests              |
| BUG-005     | CreatePlayerPageTests.cs                | 2-3 tests                                                                                                                                                                | 1-2 tests              |
| BUG-006     | ThemeRegistryTests.cs                   | 5-10 value-specific tests (hex assertions). Invariant tests (`BackgroundIsNotPureBlack`, `Surface != #FFFFFF`, `DrawerBackground == Surface`) pass WITHOUT modification. | 0                      |
| BUG-007     | None                                    | 0 (no tests assert on Color)                                                                                                                                             | 0                      |

**Total estimated:** 11-22 test updates, 5-8 new tests.

### New Test Cases Needed

**BUG-004:**

- `ActionSheetCancel_ShowsBlankForm_DoesNotNavigate`
- `TemplatePickerDialogCancel_ShowsBlankForm_DoesNotNavigate`
- `ActionSheetCancel_BackButtonStillNavigates`

**BUG-002:**

- `TemplateContext_ShowsTemplatePicker_WhenTemplatesExist`
- `TemplateContext_ShowsBlankForm_WhenNoTemplatesExist`

**BUG-005:**

- `Render_SkillRatingFieldIsEmpty_OnLoad`
- `Submit_WithNullSkillRating_UsesDefaultValue`

**BUG-003:**

- `Handle_TemplateCreation_WithNullEventId_Succeeds`

---

## 11. Tradeoffs & Alternatives Considered

### D-1: Swipe Icon Sizing

**Selected:** CSS `width/height` override → **High confidence**
**Rejected:** JS innerHTML class change to `mud-icon-size-large` — less maintainable, couples to MudBlazor internals.

### D-2: Template Picker in Settings

**Selected:** Reuse `TemplatePickerDialog` with cancel→blank-form → **High confidence**
**Rejected:** Inline button on form, MudSelect dropdown — inconsistent with existing dialog-based picker UX.

### D-3: Template Error Investigation

**Selected:** Integration-test-first diagnosis → **Low confidence** (root cause unknown)
**Rejected:** Speculative fix — too risky without understanding the actual failure.

### D-4: Bottom Sheet Cancel (P0)

**Selected:** Decouple dismiss from navigation → **High confidence**
**Rejected:** Move action sheet to GamesTabContent — would break template creation route.

### D-5: Skill Rating Optional

**Selected:** Align to nullable `int?` pattern from CreateRosterPlayerPage → **High confidence**
**Rejected:** Make CreatePlayerCommand nullable — too broad, violates AC-005-7.

### D-6: Theme Palette Redesign (Revised Round 2)

**Selected:** Standardize semantics + off-black/off-white structural colors → **Medium confidence**
**Rejected:** All themes use Apple blue Primary (destroys personality), no changes (ignores spec), pure Apple HIG values #000000/#FFFFFF (violates test invariants and AC-006-8).

### D-7: Color.Secondary Fix

**Selected:** Fix shared components only, document rest → **High confidence**
**Rejected:** Global 33-file replacement — too broad for this batch.

---

## 12. Implementation Waves & Deliverables

### Wave 1: Independent Low-Risk Fixes (Parallel)

| Bug               | Files                                                 | Risk |
| ----------------- | ----------------------------------------------------- | ---- |
| BUG-001 + BUG-008 | `list-swipe.css`                                      | 🟢   |
| BUG-005           | `CreatePlayerPage.razor.cs`, `CreatePlayerPage.razor` | 🟢   |
| BUG-007           | `EmptyState.razor`, `LoadingState.razor`              | 🟢   |

These can be implemented simultaneously by independent workers or sequentially without conflict.

### Wave 2: P0 Blocker + Coupled Fixes (Sequential)

| Order | Bug     | Files                     | Risk |
| ----- | ------- | ------------------------- | ---- |
| 1st   | BUG-004 | `CreateGamePage.razor.cs` | 🟡   |
| 2nd   | BUG-002 | `CreateGamePage.razor.cs` | 🟡   |
| 3rd   | BUG-003 | Investigation → TBD       | 🟡   |

BUG-004 must be first (fixes cancel semantics). BUG-002 depends on BUG-004 cancel fix. BUG-003 may be resolved by BUG-002/004 changes.

### Wave 3: Large Scope Theme Redesign

| Bug     | Files                                                                | Risk |
| ------- | -------------------------------------------------------------------- | ---- |
| BUG-006 | `ThemeRegistry.cs`, `ThemePreviewCard.razor`, `AppearancePage.razor` | 🟡   |

Implementation strategy: Change Professional theme first → verify tests → change remaining themes one-by-one → update PreviewSwatches → fix Color.Secondary → visual verification per theme.

---

## Acceptance Criteria Mapping

| AC ID    | Requirement                                                                                 | Design Section                                           | Status  |
| -------- | ------------------------------------------------------------------------------------------- | -------------------------------------------------------- | ------- |
| AC-001-1 | SVG icon ≥32px                                                                              | §4.1 (36px CSS override)                                 | Covered |
| AC-001-2 | Icon centered in 80px container                                                             | §4.1 (flex center, unchanged)                            | Covered |
| AC-001-3 | Fix propagates to all 5 list pages                                                          | §4.1 (shared CSS)                                        | Covered |
| AC-001-4 | Dead .delete-icon removed                                                                   | §4.1 (BUG-008 combined)                                  | Covered |
| AC-002-1 | Template picker shown in settings create                                                    | §4.3 (OnInitializedAsync change)                         | Covered |
| AC-002-2 | Template selection pre-fills form                                                           | §4.3 (existing ApplyBuiltInTemplate/ApplyCustomTemplate) | Covered |
| AC-002-3 | "Create Custom" shows blank form                                                            | §4.3 (existing dialog "Create Custom" handler)           | Covered |
| AC-002-4 | Cancel picker shows blank form                                                              | §4.2 (BUG-004 cancel fix)                                | Covered |
| AC-002-5 | Event-context unchanged                                                                     | §4.3 (only else branch modified)                         | Covered |
| AC-003-1 | CreateGameCommand null EventId succeeds                                                     | §4.4 (integration test)                                  | Covered |
| AC-003-2 | Template persists with null EventId                                                         | §4.4 (integration test)                                  | Covered |
| AC-003-3 | Submit completes, navigates to list                                                         | §4.4 (depends on investigation)                          | Covered |
| AC-003-4 | Template appears in settings list                                                           | §4.4 (depends on investigation)                          | Covered |
| AC-004-1 | Overlay tap closes sheet, shows form                                                        | §4.2 (OnActionSheetCancel fix)                           | Covered |
| AC-004-2 | Cancel button closes sheet, shows form                                                      | §4.2 (same fix)                                          | Covered |
| AC-004-3 | User can fill/submit form after dismiss                                                     | §4.2 (FlowStep.Form transition)                          | Covered |
| AC-004-4 | Dialog cancel shows blank form                                                              | §4.2 (OpenTemplatePickerDialog fix)                      | Covered |
| AC-004-5 | Back button still navigates                                                                 | §4.2 (Cancel() unchanged)                                | Covered |
| AC-004-6 | Template selection populates form                                                           | §4.2 (existing handlers unchanged)                       | Covered |
| AC-005-1 | Skill rating empty on load                                                                  | §4.5 (int? initialized to null)                          | Covered |
| AC-005-2 | Empty skill rating submits                                                                  | §4.5 (null coercion)                                     | Covered |
| AC-005-3 | Null coerces to default 50                                                                  | §4.5 (explicit ?? coercion)                              | Covered |
| AC-005-4 | Explicit value preserved                                                                    | §4.5 (unchanged data flow)                               | Covered |
| AC-005-5 | Deferred validation                                                                         | §4.5 (FormFieldState + IsTouched)                        | Covered |
| AC-005-6 | HelperText displayed                                                                        | §4.5 (HelperText parameter)                              | Covered |
| AC-005-7 | Command/Handler/Validator unchanged                                                         | §4.5 (UI-only change)                                    | Covered |
| AC-006-1 | All themes align to Apple HIG semantics                                                     | §4.6 (semantic color table)                              | Covered |
| AC-006-2 | Professional light Background = #F2F2F7                                                     | §4.6 (palette changes table)                             | Covered |
| AC-006-3 | Professional dark uses #1C1C1E (Apple elevated)                                             | §4.6 (dark Background = #1C1C1E)                         | Covered |
| AC-006-4 | PreviewSwatches match palette                                                               | §4.6 (swatch update)                                     | Covered |
| AC-006-5 | ThemePreviewCard text muted gray                                                            | §4.6 (Color.Secondary fix)                               | Covered |
| AC-006-6 | AppearancePage label muted gray                                                             | §4.6 (Color.Secondary fix)                               | Covered |
| AC-006-7 | Each theme retains distinct identity                                                        | §4.6 (Primary preserved per theme)                       | Covered |
| AC-006-8 | ThemeRegistryTests pass (no pure black bg, Surface != #FFFFFF, DrawerBackground == Surface) | §4.6 (invariant compliance tables) + §10                 | Covered |
| AC-007-1 | Full audit list documented                                                                  | §4.7 (audit table)                                       | Covered |
| AC-007-2 | Shared components fixed                                                                     | §4.7 (EmptyState + LoadingState)                         | Covered |
| AC-007-3 | Accent-intent preserved                                                                     | §4.7 (preserved table)                                   | Covered |
| AC-008-1 | Dead .delete-icon removed                                                                   | §4.1 (combined fix)                                      | Covered |
| AC-008-2 | Proper .mud-svg-icon rule exists                                                            | §4.1 (CSS with width/height)                             | Covered |
