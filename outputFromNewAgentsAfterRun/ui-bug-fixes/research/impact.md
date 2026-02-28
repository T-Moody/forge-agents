# Research: Impact

## Summary

Impact analysis covers 6 bug areas across 4 severity levels. The most critical bug is the add-game-to-event black screen/navigation issue (Bug 3), which blocks the core tournament workflow — events cannot progress past the players step. The swipe delete icon (Bug 1) and appearance colors (Bug 5) are low-risk CSS/visual-only changes with zero test breakage. Game template creation issues (Bug 2) affect the settings template library but have workarounds via event-scoped game creation. Skill rating optionality (Bug 4) requires changes to CreatePlayerPage UI only — the command/handler/validator can stay unchanged by coercing null to the default. Total estimated test impact: 5–15 existing tests need updates across 3–5 test files, plus 5–10 new tests. The codebase has 99 occurrences of `Color.Secondary` on Razor elements, but the bug report scopes the fix to the Appearance page only (2 files, 2 lines changed).

---

## Findings

### F-1: Swipe delete icon change propagates to all 5 list pages via single JS+CSS

**Category:** blast-radius

The swipe-to-delete indicator is rendered in `showDeleteIndicator()` in `list-swipe-bridge.js` (line 155). It uses a single inline SVG with `mud-icon-size-medium` (24×24px) inside an 80px-wide container. The CSS is in `list-swipe.css`. Both files are shared across all 5 list pages: EventsPage, RosterPage, GamesPage, ParticipantsTabContent, GamesTabContent.

Changing the icon size (CSS or JS) affects ALL 5 pages simultaneously. This is a LOW-RISK change because all pages share identical swipe behavior and there is no per-page customization.

**Evidence:**

- `src/Tornimate.Mobile/wwwroot/js/list-swipe-bridge.js` line 155–167
- `src/TourneyPal.UI/wwwroot/css/list-swipe.css` lines 68–93
- 5 pages implement `OnSwipeDeleteAsync`: EventsPage, RosterPage, GamesPage, ParticipantsTabContent, GamesTabContent

**Relevance:** Low blast radius. Single CSS/JS change. No .NET code changes. No test updates needed (JS/CSS only, no JS test framework).

---

### F-2: No automated tests cover swipe delete visual appearance

**Category:** testing

The swipe-to-delete visual indicator is rendered entirely in JavaScript and styled by CSS. The project has no JS test framework configured. Existing bUnit tests for list pages do NOT test swipe visuals — they test the `OnSwipeDeleteAsync` .NET callback behavior only. No tests will break from a CSS/JS icon size change.

**Evidence:**

- `docs/feature/ux-consistency-overhaul/memory/implementer-06.mem.md`: "TDD not applicable: JS/CSS-only changes with no JS test framework configured"
- `tests/TourneyPal.Tests/Features/Events/EventsPage*.cs` — tests `OnSwipeDeleteAsync` behavior, not visual indicator

**Relevance:** Zero test breakage risk. Manual visual verification required on all 5 list pages.

---

### F-3: Template creation missing dropdown — affects only settings game library flow

**Category:** blast-radius

The CreateGamePage handles two routes:

1. `/events/{EventId}/games/add` — event-scoped game creation (has template picker)
2. `/settings/game-templates/create` — template creation (skips template picker)

When `IsTemplate` is true (`EventId is null`), `OnInitializedAsync()` skips directly to `FlowStep.Form` (line 157). The template picker dialog is never shown. This means creating a new template from settings has NO way to pre-fill from an existing template.

The blast radius is limited to the settings game templates page only. Event-scoped game creation works correctly because it triggers the template picker flow.

**Evidence:**

- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor.cs` lines 117–162 (`OnInitializedAsync`)
- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor` lines 1–2 (dual routes)
- Line 156: `else { _flowStep = FlowStep.Form; }` — IsTemplate path skips picker

**Relevance:** Isolated to settings. Fix requires adding template picker to IsTemplate path. Does not affect event game creation.

---

### F-4: Template creation error — static analysis cannot reproduce

**Category:** risk

The `Submit()` flow in CreateGamePage (lines 352–436) calls `GameDataService.CreateGameAsync()` which creates a `CreateGameCommand` and sends via MediatR. `CreateGameHandler` validates dates and delegates to `GameCreationService`.

`GameCreationService.CreateAsync()` sets `EventId = null` for templates (line 31), which is correct behavior. The `CreateGameValidator` has no template-specific validation rules that would reject templates.

The error may be runtime-specific (e.g., database constraint, concurrency, or a MudBlazor form validation issue when no EventId is present). Static analysis cannot reproduce the error. Runtime debugging is needed.

**Evidence:**

- `src/TourneyPal.Core/Features/Games/GameCreationService.cs` — full file, 94 lines
- `src/TourneyPal.Common/Features/Games/CreateGame/CreateGameValidator.cs` — no template-specific rules
- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor.cs` lines 352–436 (`Submit`)

**Relevance:** Moderate impact. Users who want to build a template library are blocked, but event-scoped game creation works.

---

### F-5: Template creation bug blocks game library management but not core tournament flow

**Category:** dependency

Downstream dependencies on game templates:

1. **GamesPage** (`/settings/game-templates`) — displays template library, uses `GetGamesAsync(includeCounts: false)`
2. **CreateGamePage** (event context) — loads templates via `GameDataService.GetGamesAsync()` for picker
3. **TemplatePickerDialog** — shows built-in + custom templates for selection

If templates cannot be created:

- The template picker in event context shows only built-in templates (Chess, Checkers, etc.)
- Users can still create custom games directly in events
- The GamesPage settings section shows empty library

Game templates are a convenience feature, not a blocking dependency for the core tournament workflow.

**Evidence:**

- `src/TourneyPal.UI/Features/Games/GamesPage.razor.cs` — manages template library
- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor.cs` lines 119–128 — loads custom templates
- `src/TourneyPal.UI/Features/Games/CreateGame/TemplatePickerDialog.razor` — shows built-in + custom

**Relevance:** Low-to-moderate severity. Core tournament workflow is not blocked. Template library is a convenience feature.

---

### F-6: Test coverage for game template creation — 4 test files at risk

**Category:** testing

Test files covering game template creation flow:

1. `CreateGameHandlerTests.cs` — 5+ tests on handler logic
2. `CreateGameValidatorTests.cs` — validator tests
3. `CreateGamePageTemplateSelectionTests.cs` — 21+ tests on template picker UI flow
4. `TemplatePickerDialogTests.cs` — dialog component tests
5. `GamesPageTemplateTests.cs` — 21+ tests on game template library page
6. `GameCreationServiceTests.cs` — service-level tests
7. `MediatorGameDataServiceTests.cs` — data service tests

Total estimated tests at risk of needing updates: ~15–25 tests depending on fix scope.

**Evidence:**

- `tests/TourneyPal.Tests/Features/Games/CreateGame/` — 4 test files
- `tests/TourneyPal.Tests/Features/Games/GamesPageTemplateTests.cs` — 21+ facts
- `tests/TourneyPal.Tests/Features/Games/CreateGame/CreateGamePageTemplateSelectionTests.cs` — 21+ facts

**Relevance:** Moderate test update effort if CreateGamePage flow changes for template path.

---

### F-7: Add-game black screen is the ListActionSheet overlay — fix is page-specific

**Category:** blast-radius

The "black screen" is the ListActionSheet overlay with CSS class `.list-action-sheet-overlay` using `background: rgba(0, 0, 0, 0.4)`. This is **expected behavior** — it's a semi-transparent backdrop.

The real bug is that the action sheet cancel navigates AWAY from the page rather than just closing the sheet. `OnActionSheetCancel()` (line 219) calls `Cancel()` which navigates to `/events/{EventId}`.

ListActionSheet is used in 6+ locations: CreateGamePage, GamesPage, GamesTabContent, ParticipantsTabContent, EventsPage, RosterPage. The fix should be in CreateGamePage's `OnActionSheetCancel`, NOT in ListActionSheet itself.

**Evidence:**

- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor.cs` lines 218–222 (`OnActionSheetCancel`)
- `src/TourneyPal.UI/Common/Components/ListActionSheet.razor` — shared component, 6+ usages
- `docs/feature/ux-consistency-overhaul/memory/implementer-10.mem.md`: "CreateGamePage.OnActionSheetCancel conflates dismiss sheet with navigate away"

**Relevance:** Fix is page-specific (CreateGamePage only). ListActionSheet component itself is correct. No global blast radius.

---

### F-8: Add-game-to-event is a critical step in the event workflow

**Category:** dependency

The event workflow requires games to function:

1. Create Event → 2. Add Players → 3. **Add Game** → 4. Configure Scoring → 5. Create Bracket → 6. Play Matches → 7. View Standings

Without games, the following features are blocked:

- Bracket creation (requires a game)
- Match scoring (requires a bracket)
- Standings calculation (requires completed matches)
- Export (uses game/bracket/standings data)
- Victory banner (checks game standings)

`EventStatusUpdater.DeriveStatusForEventAsync()` returns `Draft` when `games.Count == 0`. If the "add game" flow fails, users cannot progress their event beyond adding players. **This is a HIGH-SEVERITY workflow blocker.**

**Evidence:**

- `src/TourneyPal.Core/Features/Events/EventStatusUpdater.cs` lines 31–35: `if (games.Count == 0) return EventStatus.Draft`
- `src/TourneyPal.UI/Features/Events/GamesTabContent.razor.cs` — add game button navigates to CreateGamePage
- `src/TourneyPal.Core/Features/Standings/StandingsCalculator.cs` — iterates games, needs at least one

**Relevance:** HIGH severity. Core tournament workflow is blocked if users cannot add games to events.

---

### F-9: ParticipantsTabContent proves correct action sheet cancel pattern

**Category:** pattern

ParticipantsTabContent uses ListActionSheet for "Add Player" with two actions ("Add New Player" and "Import from Roster"). When cancelled, it hides the sheet and stays on the page — it does NOT navigate away.

This proves the ListActionSheet component works correctly. The bug is exclusively in CreateGamePage's cancel handler, which conflates "dismiss action sheet" with "navigate away from page."

**Evidence:**

- `src/TourneyPal.UI/Features/Events/ParticipantsTabContent.razor.cs` — cancel hides sheet, stays on page
- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor.cs` line 219–222
- `tests/TourneyPal.Tests/Features/Events/ParticipantsTabContentTests.cs` — tests action sheet behavior

**Relevance:** Clear fix pattern: mirror ParticipantsTabContent cancel behavior.

---

### F-10: Skill rating change requires CreatePlayerPage + null-to-default coercion

**Category:** blast-radius

Making skill rating optional in CreatePlayerPage requires changes across multiple layers:

| Layer           | File                                 | Change                                                        |
| --------------- | ------------------------------------ | ------------------------------------------------------------- |
| UI field        | `CreatePlayerPage.razor.cs`          | `int _skillRating = DefaultSkillRating` → `int? _skillRating` |
| UI binding      | `CreatePlayerPage.razor`             | `MudNumericField T="int"` → `T="int?"`                        |
| Submit coercion | `CreatePlayerPage.razor.cs` Submit() | Add `_skillRating ?? DefaultSkillRating`                      |
| Command         | `CreatePlayerCommand.cs`             | Keep `int SkillRating = DefaultSkillRating` (unchanged)       |
| Interface       | `IPlayerDataService.cs`              | Already `int skillRating = DefaultSkillRating` (unchanged)    |

The `CreateRosterPlayerPage` already implements the correct pattern: `int? _skillRating` with null-to-default coercion at submission. The fix is to mirror that pattern in `CreatePlayerPage`.

**Evidence:**

- `src/TourneyPal.UI/Features/Players/CreatePlayer/CreatePlayerPage.razor.cs` line 58
- `src/TourneyPal.UI/Features/Roster/CreateRosterPlayer/CreateRosterPlayerPage.razor.cs` line 42
- `src/TourneyPal.Common/Features/Players/CreatePlayer/CreatePlayerCommand.cs` line 14
- `src/TourneyPal.Common/Features/Players/IPlayerDataService.cs` line 19

**Relevance:** Medium blast radius. UI-only change possible by coercing null to default before passing to service. Command/handler/validator stay unchanged.

---

### F-11: Skill rating tests will need updates for nullable field behavior

**Category:** testing

Test files affected by making skill rating optional:

1. **CreatePlayerPageTests.cs** — 15 tests total, 2–4 directly test skill rating rendering and submission
2. **CreatePlayerValidatorTests.cs** — `Validate_WithDefaultSkillRating_ShouldPass` unchanged if command stays the same

Estimated test updates: 2–4 tests in CreatePlayerPageTests.cs. No validator test changes if command signature stays the same.

**Evidence:**

- `tests/TourneyPal.Tests/Features/Players/CreatePlayer/CreatePlayerPageTests.cs` — 15 facts
- `tests/TourneyPal.Tests/Features/Players/CreatePlayer/CreatePlayerValidatorTests.cs`

**Relevance:** Low test update effort. Most tests work unchanged if command defaults apply.

---

### F-12: Color.Secondary on MudText is widespread — 99 usages across 33+ Razor files

**Category:** blast-radius

`Color="Color.Secondary"` appears 99 times across Razor files:

- ~65 on MudText elements (secondary text intent — likely bugs)
- ~15 on MudIcon, MudButton, MudChip (accent color intent — correct usage)
- ~5 on MudSwitch section headers

The bug report targets the Appearance page (ThemePreviewCard + AppearancePage) only. Fixing those 2 files / 2 lines is safe and isolated. A global audit would affect 33+ files across 15+ features.

In MudBlazor, `Color.Secondary` on MudText applies the Secondary palette color (e.g., `#ff4081` pink). The intended behavior is `TextSecondary` (`#6E6E80` muted gray).

**Evidence:**

- 99 occurrences of `Color="Color.Secondary"` across Razor files
- `src/TourneyPal.UI/Features/Settings/ThemePreviewCard.razor` line 9
- `src/TourneyPal.UI/Features/Settings/AppearancePage.razor` line 19
- Affected files include: EmptyState, LoadingState, TaskPage, ListActionSheet, SettingsPage, RosterPage, and 27+ more

**Relevance:** HIGH risk if done globally (33+ files). LOW risk if scoped to Appearance page only (2 files, 2 lines).

---

### F-13: No tests assert on Color.Secondary — zero test breakage from color changes

**Category:** testing

Search across all test files for `Color.Secondary` yields zero matches. The bUnit tests for ThemePreviewCard (7+ tests) and AppearancePage (7+ tests) verify content rendering, click callbacks, and theme selection — but do NOT assert on color values.

**Evidence:**

- grep for `Color.Secondary` in tests/ — 0 matches
- `tests/TourneyPal.Tests/Features/Settings/ThemePreviewCardTests.cs` — tests name/desc rendering, not color
- `tests/TourneyPal.Tests/Features/Settings/AppearancePageTests.cs` — tests toggle/theme selection, not color

**Relevance:** Zero test breakage. Manual visual verification sufficient.

---

### F-14: Cross-cutting test impact summary — estimated 5–10 test updates across all bugs

**Category:** testing

| Bug Area                     | Test Files at Risk    | Tests Needing Updates | New Tests Needed |
| ---------------------------- | --------------------- | --------------------- | ---------------- |
| Swipe icon size              | 0                     | 0                     | 0 (JS/CSS only)  |
| Template creation (dropdown) | 1–2                   | 2–5                   | 1–3              |
| Template creation (error)    | Depends on root cause | 0–5                   | 1–2              |
| Add game black screen        | 1                     | 1–3                   | 2–3              |
| Skill rating optional        | 1                     | 2–4                   | 1–2              |
| Appearance colors            | 0                     | 0                     | 0–1              |

**Total** test files likely needing changes: 3–5. Total existing tests needing updates: 5–15. Total new tests to write: 5–10.

**Evidence:**

- `tests/TourneyPal.Tests/Features/Games/CreateGame/` — 4 test files
- `tests/TourneyPal.Tests/Features/Players/CreatePlayer/CreatePlayerPageTests.cs` — 15 facts
- `tests/TourneyPal.Tests/Common/Components/ListActionSheetTests.cs` — 13 facts
- `tests/TourneyPal.Tests/Features/Settings/ThemePreviewCardTests.cs`
- `tests/TourneyPal.Tests/Features/Settings/AppearancePageTests.cs`

**Relevance:** Test impact is manageable. Most updates are in CreateGamePage and CreatePlayerPage test files.

---

### F-15: Severity ranking — add-game-to-event is highest priority

**Category:** risk

Impact severity ranking by user-facing impact:

1. **CRITICAL: Add game to event (black screen + navigate back)** — Blocks core tournament workflow. Users cannot progress beyond adding players. No obvious workaround.
2. **HIGH: Template creation error** — Prevents building template library. Workaround: create games in events with "Save as Template" toggle.
3. **MEDIUM: Skill rating pre-filled** — Incorrect UX but not blocking. Users can manually clear the field.
4. **MEDIUM: Template creation missing dropdown** — Settings-only convenience feature with workarounds.
5. **LOW: Swipe delete icon too small** — Visual polish. Delete still works correctly.
6. **LOW: Appearance colors** — Cosmetic issue on settings page. No functionality impact.

**Evidence:**

- Event workflow: Create Event → Add Players → Add Game (BLOCKED) → Create Bracket → Play
- `EventStatusUpdater` returns `Draft` with 0 games — event stays in limbo
- All other bugs have functional workarounds

**Relevance:** Bug 3 (add game) should be fixed first as it's a workflow blocker. Other bugs are convenience/polish.

---

## Source Files Examined

- `src/Tornimate.Mobile/wwwroot/js/list-swipe-bridge.js`
- `src/TourneyPal.UI/wwwroot/css/list-swipe.css`
- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor`
- `src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor.cs`
- `src/TourneyPal.UI/Features/Games/CreateGame/TemplatePickerDialog.razor`
- `src/TourneyPal.UI/Features/Games/GamesPage.razor.cs`
- `src/TourneyPal.UI/Features/Events/GamesTabContent.razor.cs`
- `src/TourneyPal.UI/Features/Events/ParticipantsTabContent.razor.cs`
- `src/TourneyPal.UI/Features/Events/EventsPage.razor.cs`
- `src/TourneyPal.UI/Features/Roster/RosterPage.razor.cs`
- `src/TourneyPal.UI/Features/Players/CreatePlayer/CreatePlayerPage.razor`
- `src/TourneyPal.UI/Features/Players/CreatePlayer/CreatePlayerPage.razor.cs`
- `src/TourneyPal.UI/Features/Roster/CreateRosterPlayer/CreateRosterPlayerPage.razor.cs`
- `src/TourneyPal.UI/Features/Settings/AppearancePage.razor`
- `src/TourneyPal.UI/Features/Settings/ThemePreviewCard.razor`
- `src/TourneyPal.UI/Common/Components/ListActionSheet.razor`
- `src/TourneyPal.Common/Features/Players/CreatePlayer/CreatePlayerCommand.cs`
- `src/TourneyPal.Common/Features/Players/IPlayerDataService.cs`
- `src/TourneyPal.Common/Features/Players/PlayerDto.cs`
- `src/TourneyPal.Common/Common/ValidationConstants.cs`
- `src/TourneyPal.Core/Features/Games/GameCreationService.cs`
- `src/TourneyPal.Core/Features/Events/EventStatusUpdater.cs`
- `src/TourneyPal.Core/Features/Standings/StandingsCalculator.cs`
- `tests/TourneyPal.Tests/Features/Players/CreatePlayer/CreatePlayerPageTests.cs`
- `tests/TourneyPal.Tests/Features/Players/CreatePlayer/CreatePlayerValidatorTests.cs`
- `tests/TourneyPal.Tests/Features/Games/CreateGame/CreateGameHandlerTests.cs`
- `tests/TourneyPal.Tests/Features/Games/CreateGame/CreateGamePageTemplateSelectionTests.cs`
- `tests/TourneyPal.Tests/Features/Games/CreateGame/TemplatePickerDialogTests.cs`
- `tests/TourneyPal.Tests/Features/Games/CreateGame/CreateGameValidatorTests.cs`
- `tests/TourneyPal.Tests/Features/Games/GamesPageTemplateTests.cs`
- `tests/TourneyPal.Tests/Features/Games/GameCreationServiceTests.cs`
- `tests/TourneyPal.Tests/Common/Components/ListActionSheetTests.cs`
- `tests/TourneyPal.Tests/Features/Settings/ThemePreviewCardTests.cs`
- `tests/TourneyPal.Tests/Features/Settings/AppearancePageTests.cs`
- `docs/feature/ux-consistency-overhaul/memory/implementer-06.mem.md`
- `docs/feature/ux-consistency-overhaul/memory/implementer-10.mem.md`

## Research Metadata

- **Confidence Level:** high
- **Coverage Estimate:** All 6 bug areas investigated with source file examination, test coverage assessment, and downstream dependency tracing. Every finding backed by file-level evidence.
- **Gaps:** Template creation runtime error (Bug 2) cannot be reproduced through static analysis — requires runtime debugging to identify root cause. The 99 occurrences of `Color.Secondary` were counted but not individually audited for correctness (some may be intentional accent usage on MudButton/MudChip/MudIcon).
