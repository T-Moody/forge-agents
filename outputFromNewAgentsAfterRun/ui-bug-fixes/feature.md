# UI Bug Fixes Batch — Feature Specification

**Slug:** `ui-bug-fixes`
**Type:** Bug Fix Batch
**Date:** 2026-02-26
**Status:** Specified

## Summary

A batch of 8 interrelated UI bugs and inconsistencies affecting swipe-to-delete icon sizing, game template management, event game creation workflow, skill rating field behavior, and theme appearance. Includes one P0 workflow blocker (games tab black screen), three P1 fixes, and three P2 polish items.

---

## Background & Context

### Research Inputs

- [research/architecture.yaml](research/architecture.yaml) — 13 findings mapping code architecture for all bug areas
- [research/impact.yaml](research/impact.yaml) — 15 findings on blast radius, severity ranking, and test impact
- [research/dependencies.yaml](research/dependencies.yaml) — 11 findings tracing full dependency chains
- [research/patterns.yaml](research/patterns.yaml) — 15 findings on existing codebase patterns and inconsistencies

### Relevant Documentation

- `docs/feature/game-system-refactor/feature.md` — Game template system design (FR-11: template route shows blank form)
- `docs/feature/ux-consistency-overhaul/memory/implementer-06.mem.md` — Swipe delete icon text removal context
- `docs/feature/ux-consistency-overhaul/memory/implementer-10.mem.md` — Known latent issue: CreateGamePage cancel conflation
- Apple Human Interface Guidelines — Color system reference

### Priority Summary

| ID      | Title                                  | Priority | Category             |
| ------- | -------------------------------------- | -------- | -------------------- |
| BUG-001 | Swipe delete icon too small            | P2       | Visual polish        |
| BUG-002 | Template picker missing from settings  | P1       | Feature enhancement  |
| BUG-003 | Template creation throws error         | P1       | Bug fix              |
| BUG-004 | Games tab black screen + navigate away | **P0**   | **Workflow blocker** |
| BUG-005 | Skill rating should be optional        | P1       | UX inconsistency     |
| BUG-006 | Theme palette redesign (Apple HIG)     | P1       | Visual identity      |
| BUG-007 | Color.Secondary audit on MudText       | P2       | UX inconsistency     |
| BUG-008 | Dead CSS rule cleanup                  | P2       | Code hygiene         |

---

## Functional Requirements

### BUG-001: Swipe Delete Icon Too Small

**Current behavior:** The swipe-to-delete indicator renders a 24×24px SVG icon (`mud-icon-size-medium`) inside an 80px-wide container via JS innerHTML in `list-swipe-bridge.js`. After the UX overhaul removed the "Delete" text label, the icon is visually undersized. A dead CSS rule `.delete-icon` exists but is never matched.

**Expected behavior:** The delete icon renders at ≥32px within the 80px container. The dead `.delete-icon` CSS is removed. A proper `.mud-svg-icon` CSS override with explicit dimensions exists.

**Affected files:**

- `src/TourneyPal.UI/wwwroot/css/list-swipe.css` — Add `.swipe-delete-indicator .mud-svg-icon` width/height, remove dead rule
- `src/Tornimate.Mobile/wwwroot/js/list-swipe-bridge.js` — Optionally update class from `mud-icon-size-medium` to `mud-icon-size-large`

**Blast radius:** LOW — Single CSS/JS change propagates to all 5 list pages (EventsPage, RosterPage, GamesPage, ParticipantsTabContent, GamesTabContent). No .NET code changes. Zero test breakage.

---

### BUG-002: Template Picker Missing from Settings Template Creation

**Current behavior:** `CreateGamePage.OnInitializedAsync()` when `IsTemplate == true` (EventId is null) sets `_flowStep = FlowStep.Form` directly, skipping the template picker. This was by design (FR-11), but the user wants the ability to build from existing templates.

**Expected behavior:** In template context, the `TemplatePickerDialog` is shown (reusing the existing component). The user can:

1. Select a system or custom template → form pre-fills with template values
2. Choose "Create Custom Game" → blank form shown
3. Cancel the picker → blank form shown (NOT navigate away)

**Affected files:**

- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor.cs` — Extend `OnInitializedAsync()` to invoke `OpenTemplatePickerDialog()` when `IsTemplate == true`

**Blast radius:** LOW — Only affects settings template creation route. Event-scoped game creation unchanged. TemplatePickerDialog requires no modifications.

---

### BUG-003: Template Creation Throws Error

**Current behavior:** Creating a game template from Settings → Game Templates → Create throws an error. Static analysis of the full chain (CreateGamePage.Submit → MediatorGameDataService → CreateGameHandler → GameCreationService → Repository) reveals no obvious validation failure for templates (EventId=null path). The root cause is unknown and requires runtime investigation.

**Expected behavior:** Template creation succeeds. Template is persisted with EventId=null. User is navigated back to the game templates list.

**Investigation strategy:**

1. Write focused integration test exercising `CreateGameCommand` with `EventId=null`
2. If handler test passes, bug is UI-side → use Playwright automation to test live on Android emulator
3. Check MudBlazor form validation behavior with null EventId
4. Check for recent schema changes adding NOT NULL constraints
5. Fix identified root cause

**Affected files:** Depends on root cause investigation. Likely candidates:

- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor.cs` (Submit flow)
- `src/TourneyPal.Core/Features/Games/CreateGame/CreateGameHandler.cs`
- `src/TourneyPal.Core/Features/Games/GameCreationService.cs`
- `src/TourneyPal.Common/Features/Games/CreateGame/CreateGameValidator.cs`

---

### BUG-004: Games Tab "Add Game" Bottom Sheet Goes Black and Navigates Away

**Current behavior:** When adding a game to an event from the Games tab (no custom templates exist):

1. User navigates to `/events/{EventId}/games/add` (CreateGamePage)
2. `ShowDecisionSheet()` immediately shows a ListActionSheet with overlay (rgba(0,0,0,0.4))
3. Screen appears dark/black with a bottom sheet
4. `OnActionSheetCancel()` calls `Cancel()` which navigates to `/events/{EventId}`
5. User is returned to the event page without creating a game

This is a **known latent issue** (implementer-10.mem.md: "conflates 'dismiss sheet' with 'navigate away'").

**Expected behavior:** Dismissing the action sheet (overlay tap or cancel button) should:

1. Close the sheet
2. Set `_flowStep = FlowStep.Form`
3. Show the blank game creation form
4. NOT navigate away

Only the page-level back button should navigate away.

**Reference pattern:** `ParticipantsTabContent.CloseAddActionSheet()` — simply sets `_addActionSheetVisible = false` without navigation.

**Affected files:**

- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor.cs` — Change `OnActionSheetCancel()` to NOT call `Cancel()`; also fix TemplatePickerDialog cancel path

**Blast radius:** LOW — Page-specific fix in CreateGamePage only. ListActionSheet component is correct. No global changes.

**Impact:** This is a **P0 workflow blocker**. Without games, events cannot progress: no bracket creation, no match scoring, no standings, no exports. EventStatusUpdater returns Draft when games.Count == 0.

---

### BUG-005: Skill Rating Should Be Optional, Not Pre-filled

**Current behavior:** `CreatePlayerPage` uses `int _skillRating = ValidationConstants.DefaultSkillRating` (50). The `MudNumericField T="int"` always shows "50". Users cannot clear the field. No HelperText, no FormFieldState deferred validation.

**Expected behavior:** Skill rating field uses `int? _skillRating` initialized to null (empty). Uses `MudNumericField<int?>` with Value/ValueChanged pattern. Shows HelperText indicating optional. Uses FormFieldState deferred validation (validate only after interaction). On submit, coerces `null` to `DefaultSkillRating` (50) at the service boundary.

**Reference pattern:** `CreateRosterPlayerPage` — already implements the optional skill rating pattern correctly:

- `int? _skillRating` (nullable, starts empty)
- `MudNumericField<int?>` with Value/ValueChanged
- `RosterValidators.ValidateSkillRating(value, Loc)` — returns null for null input
- HelperText present

**Affected files:**

- `src/TourneyPal.UI/Features/Players/CreatePlayer/CreatePlayerPage.razor.cs` — Change field type, add validation, add handler
- `src/TourneyPal.UI/Features/Players/CreatePlayer/CreatePlayerPage.razor` — Change MudNumericField binding

**Blast radius:** LOW — UI-only change. Command/Handler/Validator/Entity all remain unchanged. The `IPlayerDataService.CreatePlayerAsync` already has `int skillRating = DefaultSkillRating` default parameter. The null-to-default coercion happens at the UI submission boundary.

---

### BUG-006: Theme Palette Redesign for Apple HIG Compliance

**Current behavior:** ThemeRegistry defines 8 themes with custom hex values. Key divergences from Apple HIG in the Professional (default) theme:

| Property   | Current (Light)  | Apple HIG (Light) | Gap   |
| ---------- | ---------------- | ----------------- | ----- |
| Primary    | #594AE2 (purple) | #007AFF (blue)    | Major |
| Secondary  | #ff4081 (pink)   | #FF2D55 (pink)    | Minor |
| Tertiary   | #1ec8a5 (teal)   | #5AC8FA (teal)    | Major |
| Background | #FAFAFC          | #F2F2F7           | Minor |
| Error      | #ff3f5f          | #FF3B30           | Close |
| Success    | #3dcb6c          | #34C759           | Close |
| Warning    | #ffb545          | #FF9500           | Close |

Additionally, `Color.Secondary` on MudText in ThemePreviewCard (line 9) and AppearancePage (line 19) displays accent pink instead of muted gray TextSecondary.

**Expected behavior:**

1. All 8 themes adjusted toward Apple HIG system colors while retaining unique personalities
2. Semantic colors (Error, Success, Warning, Info) standardized across all themes to Apple HIG values
3. Background/Surface values aligned to Apple system backgrounds
4. Each theme retains its distinct Primary color identity
5. PreviewSwatches updated to match changed palette values
6. `Color.Secondary` removed from MudText in ThemePreviewCard and AppearancePage

**Affected files:**

- `src/TourneyPal.UI/Common/Theme/ThemeRegistry.cs` — Update palette hex values and PreviewSwatches for all 8 themes (708-line file)
- `src/TourneyPal.UI/Features/Settings/ThemePreviewCard.razor` — Remove `Color="Color.Secondary"` from description MudText
- `src/TourneyPal.UI/Features/Settings/AppearancePage.razor` — Remove `Color="Color.Secondary"` from mode label MudText

---

### BUG-007: Color.Secondary Misuse Audit on MudText

**Current behavior:** ~99 occurrences of `Color="Color.Secondary"` across ~33 Razor files. On MudText, this applies the accent color instead of muted text color. The intent in most cases is muted secondary text.

**Expected behavior:** Document the full audit list. Fix high-visibility shared components (EmptyState, LoadingState) as part of this batch. Leave remaining instances for a future focused task. Do NOT change `Color.Secondary` on MudIcon, MudButton, MudChip where accent color is the correct intent.

**Affected files:** Audit scope — estimated 5-10 files for high-visibility fixes beyond BUG-006.

---

### BUG-008: Dead CSS Rule .delete-icon in list-swipe.css

**Current behavior:** `.swipe-delete-indicator .delete-icon` CSS rule (font-size: 1.25rem) exists but is never matched. JS creates icons with `.mud-svg-icon`, not `.delete-icon`.

**Expected behavior:** Dead rule removed. Proper `.swipe-delete-indicator .mud-svg-icon` rule with explicit dimensions added.

**Affected files:**

- `src/TourneyPal.UI/wwwroot/css/list-swipe.css`

---

## Non-Functional Requirements

- **Performance:** No performance regression. Icon size change is CSS-only (no layout reflow). Theme changes are static hex values (no runtime computation).
- **Accessibility:** All interactive elements maintain 44×44px minimum touch targets. Color contrast ratios must meet WCAG AA for text readability after theme palette changes.
- **Offline:** All fixes work offline. No network dependencies introduced.
- **Localization:** Any new HelperText (BUG-005) must use IStringLocalizer. No hardcoded strings.
- **Backward compatibility:** Existing events, games, players, and templates must not be affected by any change.

---

## Constraints & Assumptions

- MAUI is the sole shipping target. Web projects must compile but receive no new features.
- One class per file. Code-behind only (no @code blocks).
- All form validation uses FluentValidation (backend) and FormFieldState deferred validation (frontend).
- Theme colors flow through MudBlazor's MudThemeProvider — no custom CSS color overrides.
- BUG-003 root cause is unknown; investigation may reveal the fix is trivial or complex.
- BUG-006 is a large scope change (~708 lines in ThemeRegistry); implementer should change values theme-by-theme with test verification between each.

---

## Acceptance Criteria Summary

| ID       | Criterion                                    | Method        |
| -------- | -------------------------------------------- | ------------- |
| AC-001-1 | Swipe delete SVG ≥32px                       | Inspection    |
| AC-001-2 | Icon centered in 80px container              | Inspection    |
| AC-001-3 | Fix propagates to all 5 list pages           | Demonstration |
| AC-001-4 | Dead .delete-icon CSS removed                | Inspection    |
| AC-002-1 | Template picker shown in settings create     | Test          |
| AC-002-2 | Template selection pre-fills form            | Test          |
| AC-002-3 | "Create Custom" shows blank form             | Test          |
| AC-002-4 | Cancel picker shows blank form (no nav)      | Test          |
| AC-002-5 | Event-context creation unchanged             | Test          |
| AC-003-1 | CreateGameCommand with null EventId succeeds | Test          |
| AC-003-2 | Template persists with null EventId          | Test          |
| AC-003-3 | Submit completes, navigates to list          | Demonstration |
| AC-003-4 | Template appears in settings list            | Demonstration |
| AC-004-1 | Overlay tap closes sheet, shows form         | Test          |
| AC-004-2 | Cancel button closes sheet, shows form       | Test          |
| AC-004-3 | User can fill/submit form after dismiss      | Demonstration |
| AC-004-4 | Dialog cancel shows blank form               | Test          |
| AC-004-5 | Back button still navigates correctly        | Test          |
| AC-004-6 | Template selection populates form            | Test          |
| AC-005-1 | Skill rating renders empty on load           | Test          |
| AC-005-2 | Empty skill rating submits successfully      | Test          |
| AC-005-3 | Null coerces to DefaultSkillRating (50)      | Test          |
| AC-005-4 | Explicit value is preserved                  | Test          |
| AC-005-5 | Deferred validation for out-of-range         | Test          |
| AC-005-6 | HelperText displayed                         | Inspection    |
| AC-005-7 | Command/Handler/Validator unchanged          | Inspection    |
| AC-006-1 | All themes align to Apple HIG semantics      | Inspection    |
| AC-006-2 | Professional light Background = #F2F2F7      | Test          |
| AC-006-3 | Professional dark Surface = #1C1C1E          | Test          |
| AC-006-4 | PreviewSwatches match palette values         | Test          |
| AC-006-5 | ThemePreviewCard text uses TextSecondary     | Inspection    |
| AC-006-6 | AppearancePage label uses TextSecondary      | Inspection    |
| AC-006-7 | Each theme retains distinct identity         | Inspection    |
| AC-006-8 | ThemeRegistryTests pass                      | Test          |
| AC-007-1 | Full audit list documented                   | Inspection    |
| AC-007-2 | Shared components fixed                      | Inspection    |
| AC-007-3 | Accent-intent usages preserved               | Inspection    |
| AC-008-1 | Dead .delete-icon rule removed               | Inspection    |
| AC-008-2 | Proper .mud-svg-icon rule exists             | Inspection    |

---

## Edge Cases & Error Handling

| ID   | Input/Condition                                                    | Expected Behavior                                     | Severity if Missed |
| ---- | ------------------------------------------------------------------ | ----------------------------------------------------- | ------------------ |
| EC-1 | BUG-002: No templates exist when opening picker in settings        | Skip picker, show blank form directly                 | Medium             |
| EC-2 | BUG-004: Action sheet option selected but template load fails      | Graceful error, stay on page                          | High               |
| EC-3 | BUG-005: User enters value, clears field, submits                  | Null accepted, coerced to default                     | Medium             |
| EC-4 | BUG-006: Primary color changed but PreviewSwatches not updated     | Swatches MUST match palette; tests should catch       | High               |
| EC-5 | BUG-004: Dialog cancel when custom templates exist (event context) | Show blank form, don't navigate away                  | Critical           |
| EC-6 | BUG-003: Invalid input (name >100 chars or empty)                  | Validation catches, error shown in form               | Low                |
| EC-7 | BUG-001: Very narrow screen (320px)                                | Icon fits in container without overflow               | Low                |
| EC-8 | BUG-002: User modifies pre-filled name from template               | Modified name used; new template is a separate entity | High               |

---

## User Stories / Flows

### Story 1: Casual tournament organizer adds first game to event

1. User opens an event → taps "Games" tab → taps "Add Game"
2. **Before fix:** Screen goes dark, tapping anywhere returns to event page (BUG-004)
3. **After fix:** Bottom sheet appears with "Use Template" / "Create Custom". User can cancel to see blank form, or select a template to pre-fill the form. Submitting creates the game.

### Story 2: Power user creates game template library from settings

1. User opens Settings → Game Templates → taps "+" (Create)
2. **Before fix:** Blank form only, cannot build from existing template (BUG-002). Submitting throws error (BUG-003).
3. **After fix:** Template picker appears with system and custom templates. User selects "Chess" → form pre-fills. Modifies settings → submits → template saved. Template appears in list.

### Story 3: User adds player to event

1. User opens event → Participants tab → "Add Player"
2. **Before fix:** Skill rating shows "50" pre-filled (BUG-005)
3. **After fix:** Skill rating field is empty with helper text "(optional)". User can leave empty or enter a value.

### Story 4: User browses theme options

1. User opens Settings → Appearance
2. **Before fix:** Theme description text appears in pink accent color (BUG-006, Color.Secondary misuse)
3. **After fix:** Description text appears in muted gray. Theme colors are Apple HIG-aligned. Each theme card shows correct preview swatches.

---

## Test Scenarios

| Scenario                                                                  | Verifies           | Priority |
| ------------------------------------------------------------------------- | ------------------ | -------- |
| CreateGamePage: action sheet cancel sets FlowStep.Form (no navigation)    | AC-004-1, AC-004-2 | P0       |
| CreateGamePage: dialog cancel in event context shows blank form           | AC-004-4, EC-5     | P0       |
| CreateGamePage: template context shows TemplatePickerDialog               | AC-002-1           | P1       |
| CreateGamePage: template picker cancel in settings shows blank form       | AC-002-4           | P1       |
| CreateGamePage: template selection pre-fills form values                  | AC-002-2           | P1       |
| CreateGameHandler: CreateGameCommand with EventId=null succeeds           | AC-003-1           | P1       |
| GameCreationService: template persists with null EventId                  | AC-003-2           | P1       |
| CreatePlayerPage: skill rating renders empty on load                      | AC-005-1           | P1       |
| CreatePlayerPage: submit with null skill rating passes DefaultSkillRating | AC-005-3           | P1       |
| CreatePlayerPage: deferred validation on skill rating field               | AC-005-5           | P1       |
| ThemeRegistry: Professional PaletteLight Background = #F2F2F7             | AC-006-2           | P1       |
| ThemeRegistry: Professional PaletteDark Surface = #1C1C1E                 | AC-006-3           | P1       |
| ThemeRegistry: PreviewSwatches match palette Primary, Secondary, Tertiary | AC-006-4           | P1       |
| ThemeRegistry: All themes pass 22-property non-null assertion             | AC-006-8           | P1       |
| Visual: Swipe delete icon ≥32px on all 5 list pages                       | AC-001-1, AC-001-3 | P2       |
| Visual: ThemePreviewCard description in muted gray                        | AC-006-5           | P2       |

---

## Dependencies & Risks

### Dependencies

- **TemplatePickerDialog** (existing component) — Reused by BUG-002; no modifications needed to the dialog itself
- **ListActionSheet** (shared component) — Not modified; fix is in page-level cancel handler
- **FormFieldState** helper — Reused by BUG-005 for deferred validation
- **RosterValidators.ValidateSkillRating** — Reference pattern for BUG-005 skill rating validation
- **MudBlazor v8** — Icon size classes, MudNumericField nullable binding, MudThemeProvider
- **ThemeRegistryTests** — Must be updated alongside BUG-006 palette changes

### Risks

| Risk                                                         | Likelihood | Impact | Mitigation                                                                          |
| ------------------------------------------------------------ | ---------- | ------ | ----------------------------------------------------------------------------------- |
| BUG-003 root cause is deep/complex                           | Medium     | High   | Live device testing via Playwright/ADB available. Integration test first to narrow. |
| BUG-006 theme changes break visual harmony                   | Medium     | Medium | Change values incrementally. Manual visual review per theme in light+dark.          |
| BUG-004 fix introduces regression in template selection flow | Low        | High   | Tests cover both action sheet and dialog cancel paths.                              |
| BUG-007 audit reveals more issues than expected              | Low        | Low    | Scoped to high-visibility components only in this batch.                            |
| BUG-002 + BUG-004 interact in CreateGamePage                 | Medium     | Medium | Both modify OnInitializedAsync cancel behavior. Implement together carefully.       |
