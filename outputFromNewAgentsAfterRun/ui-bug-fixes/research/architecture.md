# Research: Architecture

## Summary

Investigated 4 bug areas across the TourneyPal architecture:

1. **Swipe delete icon size**: The icon is rendered via inline SVG in `list-swipe-bridge.js` using MudBlazor's medium icon size (24x24px) within an 80px container. CSS has no size override. Fix is a single CSS or JS change affecting all 5 list pages.

2. **Game template settings**: `CreateGamePage` intentionally skips template picker for settings route (per FR-11 spec). User wants template picker available in settings too. Template creation error needs runtime reproduction — code path appears structurally sound. `TemplatePickerDialog` component already exists and can be reused.

3. **Games tab in events**: The "black screen" is the `ListActionSheet` overlay (`rgba(0,0,0,0.4)`). The "returns without creating" is caused by `OnActionSheetCancel` conflating dismiss with navigation away (known latent issue from previous UX overhaul). Skill rating discrepancy: `CreatePlayerPage` pre-fills with `DefaultSkillRating` (50) while roster uses nullable optional field.

4. **Appearance colors**: `ThemePreviewCard` and `AppearancePage` use `Color.Secondary` on `MudText`, which applies the theme's accent color (`#ff4081` pink for Professional) instead of the muted `TextSecondary` color. Fix is to remove `Color.Secondary` from text elements.

---

## Findings

### F-1: Swipe delete indicator is rendered via JS innerHTML in list-swipe-bridge.js

**Category:** structure

The swipe-to-delete icon is dynamically created in `showDeleteIndicator()` at line 155 of `list-swipe-bridge.js`. The function creates a `<div class="swipe-delete-indicator">` and sets innerHTML to an inline SVG trash icon wrapped in `<span class="mud-icon-root">`. The SVG uses `class="mud-svg-icon mud-icon-size-medium"` — this is the MudBlazor medium icon size class. There is NO explicit width/height set on the SVG or the icon root span. A previous UX overhaul (task-06) removed the "Delete" text label from the indicator, leaving only the icon. After text removal, the icon may appear small because the indicator container (80px wide) is unchanged while the icon has no explicit sizing.

**Evidence:**

- `src/Tornimate.Mobile/wwwroot/js/list-swipe-bridge.js` lines 155-167
- innerHTML uses `mud-icon-size-medium` class (24px default SVG viewBox)

**Relevance:** Fix requires adjusting CSS (`.swipe-delete-indicator .mud-svg-icon`) or the inline SVG size in the JS file.

---

### F-2: CSS for swipe-delete-indicator constrains icon display within 80px container

**Category:** structure

The `.swipe-delete-indicator` CSS in `list-swipe.css` defines:

- `width: 80px` (fixed width container)
- `display: flex; align-items: center; justify-content: center`
- The `.swipe-delete-indicator .mud-svg-icon` rule only sets `fill` color
- A `.swipe-delete-indicator .delete-icon` rule exists with `font-size: 1.25rem` but is **DEAD CSS** — the JS uses `.mud-svg-icon` class, not `.delete-icon`

The icon uses MudBlazor's default `mud-icon-size-medium` (24x24px), which is small relative to the 80px container. There's no CSS override for icon dimensions.

**Evidence:**

- `src/TourneyPal.UI/wwwroot/css/list-swipe.css` lines 68-93
- `.delete-icon` CSS class is dead code (JS uses `.mud-svg-icon`)

**Relevance:** Fix should increase SVG size via CSS (`.mud-svg-icon` width/height) or use `mud-icon-size-large` class in JS innerHTML.

---

### F-3: Swipe infrastructure is shared across all 5 list pages via single JS+CSS

**Category:** structure

The swipe-to-delete icon rendering is centralized in one JS file and one CSS file. Changes propagate to all pages: EventsPage, RosterPage, GamesPage, ParticipantsTabContent, and GamesTabContent. No per-page icon customization exists.

**Evidence:**

- `src/Tornimate.Mobile/wwwroot/js/list-swipe-bridge.js` (factory pattern, shared)
- `src/TourneyPal.UI/wwwroot/css/list-swipe.css` (shared styles)
- `docs/feature/ux-consistency-overhaul/memory/implementer-06.mem.md` confirms propagation

**Relevance:** A single CSS/JS change fixes all 5 pages simultaneously. Low risk of regression.

---

### F-4: Template creation from settings goes directly to blank form — no template picker

**Category:** structure

`CreateGamePage` handles two routes:

- `/events/{EventId:guid}/games/add` (event context) — shows template picker or action sheet
- `/settings/game-templates/create` (template context) — goes directly to blank form

In `OnInitializedAsync()` (lines 119-142), when `EventId` is null (template context), the code sets `_flowStep = FlowStep.Form` and returns. No template picker or "build from existing" dropdown is shown. The feature spec (FR-11) explicitly designed this: "template-creation route shows blank form directly (no Step 1)."

The user's bug report says "Creating a new template is missing the dropdown that allows users to build from existing templates." This means the user wants template picker available in settings too. This is a feature gap relative to the original spec, not a regression.

**Evidence:**

- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor.cs` lines 119-142
- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor` lines 1-2 (dual routes)
- `docs/feature/game-system-refactor/feature.md` lines 46, 135-136

**Relevance:** Fix requires extending the FlowStep logic to also show template picker in template context, or adding a separate dropdown/button.

---

### F-5: GamesPage (settings) navigates to CreateGamePage for template creation

**Category:** structure

The game template management page at `/settings/game-templates` (`GamesPage.razor`) has:

- Header add button calling `OpenCreateGamePage()` → navigates to `/settings/game-templates/create`
- Empty state action calling `NavigateToCreateTemplate()` → same route

Both navigate to `CreateGamePage` which currently shows a blank form with no template picker.

**Evidence:**

- `src/TourneyPal.UI/Features/Games/GamesPage.razor` lines 15-16 (header add button)
- `src/TourneyPal.UI/Features/Games/GamesPage.razor.cs` lines 274-282

**Relevance:** The entry point for template creation is well-defined. Fix affects CreateGamePage behavior when IsTemplate is true.

---

### F-6: Template creation error — Submit flow and CreateGameAsync chain

**Category:** risk

The user reports "Creating a template throws an error and doesn't work anymore." The creation chain is:

1. `CreateGamePage.Submit()` (lines 352-436) calls `GameDataService.CreateGameAsync()`
2. `MediatorGameDataService.CreateGameAsync()` creates `CreateGameCommand` and sends via MediatR
3. `CreateGameHandler.Handle()` validates scheduled dates and delegates to `GameCreationService`
4. `GameCreationService.CreateAsync()` creates the Game entity and persists via repository

For template context (EventId=null):

- No scheduled date validation occurs
- `SaveAsTemplate` logic is skipped
- Falls through to `repository.AddAsync(primaryGame)`

The FluentValidation validator (`CreateGameValidator`) does not have rules specific to template-vs-event context. The Game entity has no constraints that would reject EventId=null. Need to reproduce at runtime to identify exact error.

**Evidence:**

- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor.cs` lines 352-436 (Submit)
- `src/TourneyPal.Core/Features/Games/CreateGame/CreateGameHandler.cs` (full file)
- `src/TourneyPal.Core/Features/Games/GameCreationService.cs` (full file)
- `src/TourneyPal.Common/Features/Games/CreateGame/CreateGameValidator.cs` (no template-specific rules)

**Relevance:** Requires runtime debugging or test reproduction to pinpoint the error. The code path appears structurally sound.

---

### F-7: TemplatePickerDialog is a MudDialog with search, system/custom template groups

**Category:** structure

`TemplatePickerDialog` at `CreateGame/TemplatePickerDialog.razor(.cs)` is a MudBlazor dialog that receives `BuiltInTemplates` and `CustomTemplates` as parameters. It renders:

- A search text field with debounce
- System templates group (from `GetGameTemplatesHandler`)
- Custom templates group (user-created library games, EventId=null)
- "Create Custom Game" button at bottom

Currently only used in event context. For template creation, this dialog could be reused to offer "build from existing template" functionality.

**Evidence:**

- `src/TourneyPal.UI/Features/Games/CreateGame/TemplatePickerDialog.razor` (full file, ~95 lines)
- `src/TourneyPal.UI/Features/Games/CreateGame/TemplatePickerDialog.razor.cs` (full file, ~65 lines)

**Relevance:** This existing component can be reused for template creation from settings with minimal changes.

---

### F-8: Add game from events uses ListActionSheet overlay that covers entire screen

**Category:** structure

When user taps "Add Game" in GamesTabContent, it navigates to `/events/{EventId}/games/add` (CreateGamePage). If custom templates exist, `OpenTemplatePickerDialog()` is called immediately in `OnInitializedAsync()`. If no custom templates exist, `ShowDecisionSheet()` sets `_actionSheetVisible = true` which renders a `ListActionSheet`.

The ListActionSheet uses CSS: `.list-action-sheet-overlay` with `position: fixed; inset: 0; background: rgba(0, 0, 0, 0.4); z-index: 1400` — this covers the ENTIRE screen with a semi-transparent black overlay. The `.list-action-sheet` (z-index: 1401) slides up from bottom.

Bug description: "screen goes black; selecting takes back to games tab without creating." Two potential issues:

1. The overlay (`rgba(0,0,0,0.4)`) may appear fully black if backdrop rendering is broken
2. The action sheet cancel navigates back via `OnActionSheetCancel()` → `Cancel()` → `NavigationManager.NavigateTo()`

If the template picker dialog is shown (custom templates exist), the MudDialog overlay exhibits similar behavior. If the dialog is canceled, the code may navigate back.

**Evidence:**

- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor.cs` lines 119-142 (flow init)
- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor` lines 18-21 (ListActionSheet)
- `src/TourneyPal.UI/wwwroot/css/list-swipe.css` lines 199-211 (overlay CSS)
- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor.cs` lines 219-223 (OnActionSheetCancel)

**Relevance:** Fix may require adjusting the overlay opacity/behavior, or fixing the dialog cancel handling to not navigate away.

---

### F-9: CreateGamePage action sheet cancel navigates away unconditionally

**Category:** risk

`OnActionSheetCancel()` (line 219) hides the action sheet and calls `Cancel()`. `Cancel()` (line 356) navigates back to `/events/{EventId}`. This means pressing the overlay backdrop or the cancel button on the action sheet immediately navigates away from CreateGamePage back to the event detail page. The user never sees the game creation form.

Similarly, if the TemplatePickerDialog is canceled and no custom templates exist, it also calls `Cancel()` and navigates back.

The memory from implementer-10 explicitly notes: _"CreateGamePage.OnActionSheetCancel conflates 'dismiss sheet' with 'navigate away' — this is a latent issue but out of scope for [previous] task."_

**Evidence:**

- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor.cs` lines 219-223
- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor.cs` lines 356-366 (Cancel)
- `docs/feature/ux-consistency-overhaul/memory/implementer-10.mem.md` (latent issue note)

**Relevance:** Known latent issue. Fix should decouple action sheet dismiss from page cancellation.

---

### F-10: Skill rating is pre-filled in CreatePlayerPage but optional in CreateRosterPlayerPage

**Category:** pattern

Two player creation pages handle skill rating differently:

| Page                                       | Field Type          | Initial Value                                 | Behavior                     |
| ------------------------------------------ | ------------------- | --------------------------------------------- | ---------------------------- |
| `CreateRosterPlayerPage` (settings roster) | `int? _skillRating` | `null`                                        | Optional, empty field        |
| `CreatePlayerPage` (event players)         | `int _skillRating`  | `ValidationConstants.DefaultSkillRating` (50) | Pre-filled, always has value |

The user says "Skill rating should be optional, not pre-filled (reference roster add page)."

The `CreatePlayerCommand` defines `int SkillRating = ValidationConstants.DefaultSkillRating` (non-nullable with default 50). Making skill rating optional in `CreatePlayerPage` requires changing the field to `int?` and potentially updating the command/handler/validator to accept nullable.

**Evidence:**

- `src/TourneyPal.UI/Features/Players/CreatePlayer/CreatePlayerPage.razor.cs` line 57
- `src/TourneyPal.UI/Features/Players/CreatePlayer/CreatePlayerPage.razor` lines 28-33
- `src/TourneyPal.UI/Features/Roster/CreateRosterPlayer/CreateRosterPlayerPage.razor.cs` line 42
- `src/TourneyPal.UI/Features/Roster/CreateRosterPlayer/CreateRosterPlayerPage.razor` lines 33-37
- `src/TourneyPal.Common/Features/Players/CreatePlayer/CreatePlayerCommand.cs` line 14
- `src/TourneyPal.Common/Common/ValidationConstants.cs` line 31 (DefaultSkillRating = 50)

**Relevance:** Fix requires changing CreatePlayerPage field to nullable, and potentially updating CreatePlayerCommand to accept `int?` with default fallback.

---

### F-11: ThemePreviewCard uses Color.Secondary for description text — renders as accent color

**Category:** pattern

In `ThemePreviewCard.razor` line 9: `<MudText Typo="Typo.body2" Color="Color.Secondary">`. In MudBlazor, `Color.Secondary` on `MudText` applies the theme's Secondary palette color as the text color. For the "Professional" theme, Secondary = `#ff4081` (pinkish-red). This means the theme description text renders in red/pink, not the muted gray that `TextSecondary` palette color provides.

The correct approach for muted secondary text in MudBlazor is to either:

1. NOT set `Color` at all and use CSS `color: var(--mud-palette-text-secondary)`, or
2. Use `Style="color: var(--mud-palette-text-secondary)"`, or
3. Remove the `Color` parameter (MudText defaults to inherit)

This same `Color.Secondary` misuse appears in `AppearancePage.razor` line 19 for the "Dark/Light" mode text.

**Evidence:**

- `src/TourneyPal.UI/Features/Settings/ThemePreviewCard.razor` line 9
- `src/TourneyPal.UI/Features/Settings/AppearancePage.razor` line 19
- `src/TourneyPal.UI/Common/Theme/ThemeRegistry.cs` line 73 (Professional: Secondary = `#ff4081`)

**Relevance:** Fix is straightforward: replace `Color.Secondary` with no Color parameter or explicit TextSecondary style.

---

### F-12: ThemeRegistry defines 8 themes with explicit palette colors including Secondary

**Category:** structure

`ThemeRegistry.cs` contains 8 theme definitions, each with PaletteLight and PaletteDark. The Secondary color varies per theme:

- Professional: `#ff4081` (pink-red)
- Cyber-Future: `#00BCD4` (cyan)
- Others vary

The `ThemePreviewCard` renders 4 color swatches from `PreviewSwatches`, which include the Secondary color as the second swatch. The swatches correctly use `GetSwatchColor()` which resolves light/dark variants. The swatches display as small colored circles and are fine.

The issue is specifically about **text elements** using `Color.Secondary` which applies the accent color to text instead of using the muted text color.

**Evidence:**

- `src/TourneyPal.UI/Common/Theme/ThemeRegistry.cs` lines 18-28 (8 theme registrations)
- `src/TourneyPal.UI/Common/Theme/ThemeRegistry.cs` lines 70-73 (Professional palette)
- `src/TourneyPal.UI/Common/Theme/ThemeDefinition.cs` (ThemeDefinition record)

**Relevance:** Bug is in UI components using `Color.Secondary` on text, not in theme definitions.

---

### F-13: Color.Secondary vs TextSecondary — widespread pattern across codebase

**Category:** pattern

MudBlazor's `Color.Secondary` enum value maps to the theme's Secondary palette color (an accent color like pink, cyan, etc.). For muted secondary text, the theme provides `TextSecondary` (e.g., `#6E6E80` for Professional light mode). These map to different CSS variables:

| Usage                        | CSS Variable                   | Professional Light Value |
| ---------------------------- | ------------------------------ | ------------------------ |
| `Color.Secondary` on MudText | `--mud-palette-secondary`      | `#ff4081` (pink)         |
| TextSecondary palette        | `--mud-palette-text-secondary` | `#6E6E80` (muted gray)   |

The codebase uses `Color.Secondary` on `MudText` components extensively across many pages. In most contexts, the intent is "muted secondary text" not "pink accent text." This is a systemic pattern issue, but the bug report specifically targets the Appearance page.

**Evidence:**

- MudBlazor: Color enum maps to palette accent colors
- Professional theme: `Secondary=#ff4081`, `TextSecondary=#6E6E80` (very different)
- Multiple pages use `Color.Secondary` on MudText for muted text intent

**Relevance:** The Appearance page fix should use TextSecondary instead. A broader audit may be needed but is out of scope for this bug batch.

---

## Source Files Examined

- `src/Tornimate.Mobile/wwwroot/js/list-swipe-bridge.js`
- `src/TourneyPal.UI/wwwroot/css/list-swipe.css`
- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor`
- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor.cs`
- `src/TourneyPal.UI/Features/Games/CreateGame/TemplatePickerDialog.razor`
- `src/TourneyPal.UI/Features/Games/CreateGame/TemplatePickerDialog.razor.cs`
- `src/TourneyPal.UI/Features/Games/GamesPage.razor`
- `src/TourneyPal.UI/Features/Games/GamesPage.razor.cs`
- `src/TourneyPal.UI/Features/Events/GamesTabContent.razor`
- `src/TourneyPal.UI/Features/Events/GamesTabContent.razor.cs`
- `src/TourneyPal.UI/Features/Settings/AppearancePage.razor`
- `src/TourneyPal.UI/Features/Settings/AppearancePage.razor.cs`
- `src/TourneyPal.UI/Features/Settings/ThemePreviewCard.razor`
- `src/TourneyPal.UI/Features/Settings/ThemePreviewCard.razor.cs`
- `src/TourneyPal.UI/Common/Components/ListActionSheet.razor`
- `src/TourneyPal.UI/Common/Components/ListActionSheet.razor.cs`
- `src/TourneyPal.UI/Common/Theme/ThemeRegistry.cs`
- `src/TourneyPal.UI/Common/Theme/ThemeDefinition.cs`
- `src/TourneyPal.UI/Common/Theme/ThemePreviewSwatch.cs`
- `src/TourneyPal.UI/Common/Theme/IAppearanceService.cs`
- `src/TourneyPal.UI/Features/Players/CreatePlayer/CreatePlayerPage.razor`
- `src/TourneyPal.UI/Features/Players/CreatePlayer/CreatePlayerPage.razor.cs`
- `src/TourneyPal.UI/Features/Roster/CreateRosterPlayer/CreateRosterPlayerPage.razor`
- `src/TourneyPal.UI/Features/Roster/CreateRosterPlayer/CreateRosterPlayerPage.razor.cs`
- `src/TourneyPal.UI/Features/Games/GameParticipants/ManagePlayersPage.razor`
- `src/TourneyPal.Common/Features/Games/CreateGame/CreateGameCommand.cs`
- `src/TourneyPal.Common/Features/Games/CreateGame/CreateGameValidator.cs`
- `src/TourneyPal.Common/Features/Games/CreateGame/CreateGameResponse.cs`
- `src/TourneyPal.Common/Features/Games/IGameDataService.cs`
- `src/TourneyPal.Common/Features/Players/CreatePlayer/CreatePlayerCommand.cs`
- `src/TourneyPal.Common/Common/ValidationConstants.cs`
- `src/TourneyPal.Core/Features/Games/CreateGame/CreateGameHandler.cs`
- `src/TourneyPal.Core/Features/Games/GameCreationService.cs`
- `src/TourneyPal.Core/Features/Games/Game.cs`
- `docs/feature/game-system-refactor/feature.md`
- `docs/feature/ux-consistency-overhaul/memory/implementer-06.mem.md`
- `docs/feature/ux-consistency-overhaul/memory/implementer-10.mem.md`

## Research Metadata

- **Confidence Level:** high
- **Coverage Estimate:** Comprehensive coverage of all 4 bug areas. All relevant source files, CSS, JS, components, handlers, commands, and validators examined.
- **Gaps:** Template creation error (F-6) could not be root-caused from static analysis alone — requires runtime reproduction. The broader `Color.Secondary` audit across the entire codebase (F-13) was noted but not fully enumerated as it's out of scope for this bug batch.
