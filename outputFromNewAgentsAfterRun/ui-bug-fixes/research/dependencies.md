# Research: Dependencies

## Summary

Traced full dependency chains for all 5 UI bugs. Key findings:

1. **Swipe icon** is a pure JS+CSS chain — no .NET dependencies, single change propagates to all 5 list pages.
2. **Template creation** has a FlowStep logic gap (no picker for template context) + a runtime creation error that needs debugging.
3. **Games tab "black screen"** is the ListActionSheet overlay (expected) but cancel handler incorrectly navigates away instead of dismissing.
4. **Skill rating** fix is UI-only — mirror the working CreateRosterPlayerPage pattern. Downstream chain unaffected.
5. **Appearance colors** route through ThemeRegistry → MudThemeProvider pipeline. All colors are palette-driven — no custom CSS overrides to chase.

---

## Findings

### F-1: Swipe delete icon rendering chain: JS innerHTML → CSS class → MudBlazor icon size

**Category:** dependency

The swipe delete icon is rendered via a three-layer dependency chain:

1. **JS (source of icon HTML):** `list-swipe-bridge.js` line 161 — `showDeleteIndicator()` creates a `<div class="swipe-delete-indicator">` and sets innerHTML to:

   ```html
   <span class="mud-icon-root"
     ><svg viewBox="0 0 24 24" class="mud-svg-icon mud-icon-size-medium">
       <path d="M6 19c0 1.1.9 2 2 2h8c1.1..." /></svg
   ></span>
   ```

   The SVG path is the Material Design delete (trash) icon inlined directly.

2. **CSS (layout + sizing):** `list-swipe.css` lines 68–93:
   - `.swipe-delete-indicator`: `width: 80px; display: flex; align-items: center; justify-content: center`
   - `.swipe-delete-indicator .mud-svg-icon`: only sets `fill` color — **NO width/height override**
   - `.swipe-delete-indicator .delete-icon`: dead CSS (`font-size: 1.25rem`) — JS uses `.mud-svg-icon`, not `.delete-icon`

3. **MudBlazor icon system (implicit):** `mud-icon-size-medium` = 24×24px. Icon is small because the text label was removed in a previous UX overhaul, but the 80px container and 24px icon were not adjusted.

**No Blazor/.NET dependency** — this is purely JS + CSS.

**Evidence:**

- `src/Tornimate.Mobile/wwwroot/js/list-swipe-bridge.js` lines 155–170
- `src/TourneyPal.UI/wwwroot/css/list-swipe.css` lines 68–93
- MudBlazor `mud-icon-size-medium` class = 24×24px SVG

**Relevance:** Fix involves either changing the CSS class in JS innerHTML (e.g., `mud-icon-size-large`) or adding a CSS `width`/`height` override for `.swipe-delete-indicator .mud-svg-icon`. No .NET code changes needed.

---

### F-2: Swipe delete propagation: single JS+CSS change affects all 5 list pages

**Category:** dependency

The `list-swipe-bridge.js` is loaded once globally by MAUI. All 5 pages share the same `showDeleteIndicator()`:

- EventsPage
- RosterPage
- GamesPage (settings)
- ParticipantsTabContent (event tab)
- GamesTabContent (event tab)

Each page initializes with `window.listSwipeBridge.create(options)`. The `showDeleteIndicator()` function is internal to the factory closure — not configurable per-page.

**Evidence:**

- `src/Tornimate.Mobile/wwwroot/js/list-swipe-bridge.js` — factory pattern
- `src/TourneyPal.UI/wwwroot/css/list-swipe.css` — single stylesheet

**Relevance:** One CSS/JS change fixes all 5 pages. No per-page work needed.

---

### F-3: Game template creation full dependency chain

**Category:** dependency

Complete chain for template creation:

| Step              | Component                | File                                | Key Detail                                                          |
| ----------------- | ------------------------ | ----------------------------------- | ------------------------------------------------------------------- |
| 1. Entry          | GamesPage                | `GamesPage.razor.cs` L274–282       | Navigates to `/settings/game-templates/create`                      |
| 2. UI Page        | CreateGamePage           | `CreateGamePage.razor.cs` L117–162  | `IsTemplate == true` → `_flowStep = FlowStep.Form` (skips picker)   |
| 3. Submit         | CreateGamePage.Submit()  | `CreateGamePage.razor.cs` L370–450  | Calls `GameDataService.CreateGameAsync(EventId: null)`              |
| 4. DataService    | MediatorGameDataService  | `MediatorGameDataService.cs` L32–68 | Builds `CreateGameCommand`, sends via MediatR                       |
| 5. Validation     | CreateGameValidator      | `CreateGameValidator.cs`            | Name required, max 100. No template-specific rules.                 |
| 6. Handler        | CreateGameHandler        | `CreateGameHandler.cs` L22–57       | Date validation skipped when EventId is null. Delegates to service. |
| 7. Domain Service | GameCreationService      | `GameCreationService.cs` L17–88     | Creates Game entity with `EventId = null`, calls `AddAsync`         |
| 8. Repository     | GameRepository → EF Core | —                                   | SQLite persistence                                                  |

**Bug analysis:** Static analysis shows no validation failure for templates. The "creation throws error" may be a runtime issue. The missing dropdown is a FlowStep logic gap — `OnInitializedAsync()` skips the template picker when `IsTemplate == true`.

**Evidence:**

- All files listed in the table above

**Relevance:** Two sub-bugs: (1) Missing template picker requires extending `OnInitializedAsync`. (2) Creation error needs runtime debugging.

---

### F-4: Template picker missing in template context — FlowStep logic gap

**Category:** dependency

`CreateGamePage.OnInitializedAsync()` has two branches:

**Event context (`EventId.HasValue == true`):**

```
1. Load built-in + custom templates
2. If custom templates exist → open TemplatePickerDialog immediately
3. If no custom templates → show ListActionSheet ("Use Template" / "Create Custom")
```

**Template context (`IsTemplate == true`):**

```
1. Load built-in + custom templates
2. _flowStep = FlowStep.Form → skip directly to blank form (line 157)
```

The `TemplatePickerDialog` component is already reusable — it accepts `BuiltInTemplates` and `CustomTemplates` as parameters and works correctly. It just isn't invoked in the template context.

**Evidence:**

- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor.cs` lines 148–162
- `src/TourneyPal.UI/Features/Games/CreateGame/TemplatePickerDialog.razor.cs` (full file)

**Relevance:** No new dependencies needed. TemplatePickerDialog is reusable. Fix is purely a logic change in `OnInitializedAsync`.

---

### F-5: Black screen dependency: ListActionSheet overlay → CSS → Cancel flow

**Category:** dependency

The "black screen" when adding a game from events is the ListActionSheet overlay:

**Component chain:**

1. GamesTabContent → "Add Game" → navigates to `/events/{EventId}/games/add`
2. `CreateGamePage.OnInitializedAsync()` → no custom templates → `ShowDecisionSheet()`
3. `ShowDecisionSheet()` sets `_actionSheetVisible = true`
4. `CreateGamePage.razor` line 18: `<ListActionSheet IsVisible="_actionSheetVisible" ...>`
5. `ListActionSheet.razor`: renders `.list-action-sheet-overlay` + `.list-action-sheet`

**CSS:**

- `.list-action-sheet-overlay`: `position: fixed; inset: 0; background: rgba(0,0,0,0.4); z-index: 1400`
- `.list-action-sheet`: `position: fixed; bottom: 0; z-index: 1401`

**The real bug — cancel conflation:**

- `ListActionSheet.HandleCancel()` → invokes `OnCancel` EventCallback
- `CreateGamePage` binds: `OnCancel="OnActionSheetCancel"`
- `OnActionSheetCancel()` (line 219): hides action sheet, then calls `Cancel()`
- `Cancel()` (line 356): **navigates to `/events/{EventId}`** — leaves the page

Tapping the backdrop or cancel button navigates back to the event page. The user never gets to see the game creation form.

**Evidence:**

- `src/TourneyPal.UI/Common/Components/ListActionSheet.razor` line 6
- `src/TourneyPal.UI/Common/Components/ListActionSheet.razor.cs` lines 48–51
- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor.cs` line 219, lines 356–365
- `src/TourneyPal.UI/wwwroot/css/list-swipe.css` lines 199–207

**Relevance:** Fix: `OnActionSheetCancel` should hide the sheet and fall through to `FlowStep.Form`, NOT call `Cancel()`. The ListActionSheet component is correct.

---

### F-6: Participants tab uses ListActionSheet without cancel-navigation — working reference

**Category:** pattern

The ParticipantsTabContent uses ListActionSheet for long-press context menus. When dismissed, it simply sets `_actionSheetVisible = false` — does NOT navigate away.

In all other 5+ locations, ListActionSheet cancel just dismisses. `CreateGamePage` is the **only** place where cancel navigates.

**Evidence:**

- `src/TourneyPal.UI/Features/Events/ParticipantsTabContent` — cancel just hides
- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor.cs` line 219 — cancel navigates away

**Relevance:** ParticipantsTabContent is the correct reference pattern.

---

### F-7: Skill rating dependency chain: UI field → DataService → Command → Validator → Handler → Entity

**Category:** dependency

**Current state in CreatePlayerPage (the bug):**

| Layer       | File                               | Type                                   | Default                   |
| ----------- | ---------------------------------- | -------------------------------------- | ------------------------- |
| UI field    | `CreatePlayerPage.razor.cs` L58    | `int _skillRating`                     | `DefaultSkillRating` (50) |
| Razor       | `CreatePlayerPage.razor` L28–32    | `MudNumericField T="int"`              | Always shows 50           |
| Submit      | `CreatePlayerPage.razor.cs` L130   | passes `_skillRating` directly         | —                         |
| Interface   | `IPlayerDataService.cs` L19        | `int skillRating = DefaultSkillRating` | 50                        |
| DataService | `MediatorPlayerDataService.cs` L31 | passes to Command                      | —                         |
| Command     | `CreatePlayerCommand.cs` L14       | `int SkillRating = DefaultSkillRating` | 50                        |
| Validator   | `CreatePlayerValidator.cs` L25–27  | `InclusiveBetween(1, 100)`             | —                         |
| Entity      | Player.SkillRating                 | `int` (non-nullable)                   | —                         |

**Working reference — CreateRosterPlayerPage:**

| Layer    | File                                  | Type                             | Default              |
| -------- | ------------------------------------- | -------------------------------- | -------------------- |
| UI field | `CreateRosterPlayerPage.razor.cs` L44 | `int? _skillRating`              | `null` (empty field) |
| Razor    | `CreateRosterPlayerPage.razor` L33–36 | `MudNumericField int?`           | Empty                |
| Submit   | L142                                  | passes `_skillRating` (nullable) | —                    |

**Evidence:**

- All files listed in tables above
- `src/TourneyPal.Common/Common/ValidationConstants.cs` line 31

**Relevance:** Fix pattern: change `int _skillRating` → `int? _skillRating`, use nullable MudNumericField, coerce `null → DefaultSkillRating` at Submit. Downstream chain unaffected.

---

### F-8: Downstream skill rating consumers unaffected by making UI field optional

**Category:** dependency

`IPlayerDataService.CreatePlayerAsync` already has default parameter `int skillRating = DefaultSkillRating`. The command also defaults. If the UI passes `_skillRating ?? DefaultSkillRating`, downstream sees the exact same value.

No impact on: CreatePlayerHandler, Player entity, Team generation, PlayerSkillRatingEditor, or existing tests.

**Evidence:**

- `src/TourneyPal.Common/Features/Players/IPlayerDataService.cs` line 19
- `src/TourneyPal.Common/Features/Players/CreatePlayer/CreatePlayerCommand.cs` line 14

**Relevance:** Pure UI change. No downstream breakage.

---

### F-9: Theme color dependency chain: ThemeRegistry → ThemeDefinition → MudTheme → MudThemeProvider → all UI

**Category:** dependency

The appearance color system:

1. **ThemeRegistry** (static, 708 lines): 8 themes, each with full `MudTheme` (PaletteLight + PaletteDark). Colors are hex strings.

2. **ThemeDefinition** (immutable record): Id, NameKey, DescriptionKey, Theme, PreviewSwatches

3. **IAppearanceService** (interface):
   - `CurrentMudTheme` → resolved from ThemeRegistry
   - Two implementations: `AppearanceService` (localStorage), `MauiAppearanceService` (Preferences API)

4. **MainLayout** (MAUI): `<MudThemeProvider Theme="@Theme" IsDarkMode="IsDarkMode" />`
   - `Theme => AppearanceService.CurrentMudTheme`

5. **AppearancePage** (`/settings/appearance`):
   - Dark/light toggle + theme card selection
   - `ThemePreviewCard` shows color swatches from `PreviewSwatches`
   - Selection → `SetThemeAsync()` → `AppearanceChanged` event → MainLayout re-renders

**Evidence:**

- `src/TourneyPal.UI/Common/Theme/ThemeRegistry.cs` (8 Build\*Theme methods)
- `src/TourneyPal.UI/Common/Theme/ThemeDefinition.cs`
- `src/TourneyPal.UI/Common/Theme/IAppearanceService.cs`
- `src/Tornimate.Mobile/Components/Layout/MainLayout.razor` line 6
- `src/TourneyPal.UI/Features/Settings/AppearancePage.razor`

**Relevance:** Fix is purely in ThemeRegistry hex values. No structural changes needed.

---

### F-10: No custom CSS color overrides — all colors flow through MudTheme palette

**Category:** dependency

The app consistently uses MudBlazor's `Color` enum in components and CSS variables (`var(--mud-palette-*)`) in stylesheets. The only hardcoded colors are:

- `.list-action-sheet-overlay`: `rgba(0, 0, 0, 0.4)` (hardcoded black opacity — intentional)
- ThemeRegistry: hex values in palettes (the source of truth)

No rogue inline `style="color: #xxx"` exists. Changing ThemeRegistry palette values propagates everywhere.

**Evidence:**

- `src/TourneyPal.UI/wwwroot/css/list-swipe.css` — uses `var(--mud-palette-*)` variables
- Razor files use `Color.Primary`, `Color.Error`, etc.

**Relevance:** Clean architecture. Updating ThemeRegistry hex values is sufficient.

---

### F-11: Professional theme palette values vs Apple HIG system colors

**Category:** dependency

Current "professional" theme PaletteLight vs Apple HIG:

| Property   | Current          | Apple HIG        |
| ---------- | ---------------- | ---------------- |
| Primary    | #594AE2 (purple) | #007AFF (blue)   |
| Secondary  | #ff4081 (pink)   | #FF2D55 (pink)   |
| Tertiary   | #1ec8a5 (teal)   | #5AC8FA (teal)   |
| Error      | #ff3f5f          | #FF3B30 (red)    |
| Success    | #3dcb6c          | #34C759 (green)  |
| Warning    | #ffb545          | #FF9500 (orange) |
| Info       | #4a86ff          | #007AFF (blue)   |
| Background | #FAFAFC          | #F2F2F7          |
| Surface    | #F2F2F6          | #F2F2F7          |

Surface/Background are already close. Primary/Secondary/Tertiary diverge significantly.

**Evidence:**

- `src/TourneyPal.UI/Common/Theme/ThemeRegistry.cs` lines 69–100 (PaletteLight), 108–136 (PaletteDark)
- Apple HIG color specifications

**Relevance:** To "follow Apple-like standards", update hex values in ThemeRegistry or add a new Apple-inspired theme. PreviewSwatches must match.

---

## Source Files Examined

- `src/Tornimate.Mobile/wwwroot/js/list-swipe-bridge.js`
- `src/TourneyPal.UI/wwwroot/css/list-swipe.css`
- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor`
- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor.cs`
- `src/TourneyPal.UI/Features/Games/CreateGame/TemplatePickerDialog.razor`
- `src/TourneyPal.UI/Features/Games/CreateGame/TemplatePickerDialog.razor.cs`
- `src/TourneyPal.UI/Features/Games/GamesPage.razor`
- `src/TourneyPal.UI/Common/Components/ListActionSheet.razor`
- `src/TourneyPal.UI/Common/Components/ListActionSheet.razor.cs`
- `src/TourneyPal.UI/Features/Players/CreatePlayer/CreatePlayerPage.razor`
- `src/TourneyPal.UI/Features/Players/CreatePlayer/CreatePlayerPage.razor.cs`
- `src/TourneyPal.UI/Features/Roster/CreateRosterPlayer/CreateRosterPlayerPage.razor`
- `src/TourneyPal.UI/Features/Roster/CreateRosterPlayer/CreateRosterPlayerPage.razor.cs`
- `src/TourneyPal.UI/Features/Settings/AppearancePage.razor`
- `src/TourneyPal.UI/Features/Settings/AppearancePage.razor.cs`
- `src/TourneyPal.UI/Features/Settings/ThemePreviewCard.razor`
- `src/TourneyPal.UI/Features/Settings/ThemePreviewCard.razor.cs`
- `src/TourneyPal.UI/Common/Theme/ThemeRegistry.cs`
- `src/TourneyPal.UI/Common/Theme/ThemeDefinition.cs`
- `src/TourneyPal.UI/Common/Theme/IAppearanceService.cs`
- `src/TourneyPal.UI/Common/Theme/AppearanceService.cs`
- `src/Tornimate.Mobile/Services/MauiAppearanceService.cs`
- `src/Tornimate.Mobile/Components/Layout/MainLayout.razor`
- `src/Tornimate.Mobile/Components/Layout/MainLayout.razor.cs`
- `src/TourneyPal.Common/Features/Games/IGameDataService.cs`
- `src/TourneyPal.Common/Features/Games/CreateGame/CreateGameCommand.cs`
- `src/TourneyPal.Common/Features/Games/CreateGame/CreateGameValidator.cs`
- `src/TourneyPal.Core/Features/Games/CreateGame/CreateGameHandler.cs`
- `src/TourneyPal.Core/Features/Games/GameCreationService.cs`
- `src/TourneyPal.Core/Features/Games/MediatorGameDataService.cs`
- `src/TourneyPal.Common/Features/Players/IPlayerDataService.cs`
- `src/TourneyPal.Common/Features/Players/CreatePlayer/CreatePlayerCommand.cs`
- `src/TourneyPal.Common/Features/Players/CreatePlayer/CreatePlayerValidator.cs`
- `src/TourneyPal.Core/Features/Players/MediatorPlayerDataService.cs`
- `src/TourneyPal.Common/Common/ValidationConstants.cs`

## Research Metadata

- **Confidence Level:** high
- **Coverage Estimate:** Comprehensive — traced every dependency chain from UI through to persistence for all 5 bugs. Read actual source files at relevant line ranges.
- **Gaps:** The game template "creation throws error" could not be root-caused via static analysis alone — the CQRS chain appears valid. Runtime debugging (e.g., EF Core exception, database constraint) may be needed to identify the actual error.
