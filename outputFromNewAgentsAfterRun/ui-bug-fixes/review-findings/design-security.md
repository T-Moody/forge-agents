# Adversarial Review: design — claude-opus-4.6

## Review Metadata

- **Review Focus:** security
- **Risk Level:** 🟡
- **Round:** 1
- **Run ID:** 2026-02-26T14:30:00Z

## Overall Verdict

**Verdict:** approve
**Confidence:** High

## Findings

### Finding 1: innerHTML usage in list-swipe-bridge.js is hardcoded SVG — no XSS vector

- **Severity:** Minor
- **Category:** security
- **Description:** The `list-swipe-bridge.js` line 160–161 uses `indicator.innerHTML` to inject an SVG delete icon. The design (BUG-001) correctly chose a CSS-only fix (width/height override) rather than modifying this JS. The innerHTML content is a hardcoded SVG string literal with no user-controlled input interpolated — the path data (`M6 19c0 1.1...`) and class names (`mud-svg-icon mud-icon-size-medium`) are all static constants. There is **no XSS vector** here. However, the design does not explicitly call out _why_ the innerHTML is safe (i.e., no dynamic data), which could lead a future implementer to mistakenly add user-sourced data to this template without sanitization.
- **Affected artifacts:** [list-swipe-bridge.js](src/Tornimate.Mobile/wwwroot/js/list-swipe-bridge.js#L160-L161), design.md §4.1 (Decision D-1)
- **Recommendation:** Add a code comment near the innerHTML assignment noting that the SVG content must remain a static literal — no user-supplied data should ever be interpolated into this template string. This is a documentation suggestion, not a blocking concern.
- **Evidence:** Source code at `list-swipe-bridge.js:160-161` shows `indicator.innerHTML = '<span class="mud-icon-root"><svg viewBox="0 0 24 24" class="mud-svg-icon mud-icon-size-medium"><path d="M6 19c0..."/></svg></span>';` — fully static, no variables or template literals.

### Finding 2: Template picker exposes only game configuration data — no sensitive information leakage

- **Severity:** Minor
- **Category:** security
- **Description:** BUG-002 extends the `TemplatePickerDialog` to the settings template creation context. The dialog receives `IReadOnlyList<GameTemplateDto>` (built-in) and `IReadOnlyList<GameDto>` (custom). These DTOs contain game configuration data (name, description, max players, scoring settings) — not user PII, credentials, or sensitive event data. The template data shown in the picker is the same data the user already created or that ships as built-in system templates. Reusing the existing dialog in a new context does not expand the data surface area.
- **Affected artifacts:** [TemplatePickerDialog.razor.cs](src/TourneyPal.UI/Features/Games/CreateGame/TemplatePickerDialog.razor.cs#L24-L30), design-output.yaml BUG-002 changes
- **Recommendation:** No action needed. The data exposed is non-sensitive and already accessible to the authenticated local user.
- **Evidence:** `TemplatePickerDialog.razor.cs` parameters: `IReadOnlyList<GameTemplateDto> BuiltInTemplates` and `IReadOnlyList<GameDto> CustomTemplates` — these are configuration DTOs without PII or auth tokens.

### Finding 3: BUG-004 cancel fix eliminates a potential state confusion vector — not a security issue

- **Severity:** Minor
- **Category:** security
- **Description:** The BUG-004 fix decouples action sheet cancel from page navigation. The current broken behavior (cancel → navigate away) is a UX bug, not a security vulnerability. The proposed fix (cancel → show blank form) follows the proven `ParticipantsTabContent.CloseAddActionSheet()` pattern. There is no authentication bypass, privilege escalation, or data exposure from either the current or proposed cancel behavior. The `Cancel()` method uses `NavigationManager.NavigateTo()` with hardcoded route strings (`/events/{EventId}`, `/settings/game-templates`) — no user-controlled navigation targets.
- **Affected artifacts:** [CreateGamePage.razor.cs](src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor.cs#L218-L221) (OnActionSheetCancel), [lines 247-258](src/TourneyPal.UI/Features/Games/CreateGame/CreateGamePage.razor.cs#L247-L258) (OpenTemplatePickerDialog cancel path)
- **Recommendation:** No action needed from a security perspective. The fix improves UX correctness without security implications.
- **Evidence:** `NavigationManager.NavigateTo($"/events/{EventId}")` and `NavigationManager.NavigateTo("/settings/game-templates")` use only server-controlled route values. `EventId` is a `Guid?` parameter, not user-supplied free text.

### Finding 4: Nullable skill rating (BUG-005) cannot bypass game matching rules

- **Severity:** Minor
- **Category:** security
- **Description:** BUG-005 changes `_skillRating` from `int` (default 50) to `int?` (default null) in the UI layer only. The null-to-default coercion (`_skillRating ?? ValidationConstants.DefaultSkillRating`) happens at the UI submission boundary before the value reaches `IPlayerDataService.CreatePlayerAsync`. The `CreatePlayerCommand`, `CreatePlayerValidator`, and `CreatePlayerHandler` remain unchanged (AC-005-7). The validator still enforces the 1–100 range. A null skill rating simply coerces to the default (50), which is the same value that was always pre-filled before this change. This means game matching rules (which operate on the persisted integer skill rating) see no behavior difference between the old and new flow.
- **Affected artifacts:** design-output.yaml BUG-005 changes, spec-output.yaml AC-005-3, AC-005-7
- **Recommendation:** No action needed. The coercion is correctly placed at the UI boundary, and backend validation remains unchanged. The design explicitly preserves the invariant that only valid integer skill ratings reach the domain layer.
- **Evidence:** Design specifies: "Submit: passes `_skillRating ?? ValidationConstants.DefaultSkillRating` to service." AC-005-7 confirms "Command/Handler/Validator unchanged." The `IPlayerDataService.CreatePlayerAsync` already has `int skillRating = DefaultSkillRating` default parameter as a safety net.

### Finding 5: Theme palette changes use static hex values — no runtime injection risk

- **Severity:** Minor
- **Category:** security
- **Description:** BUG-006 modifies `ThemeRegistry.cs` with new hex color values. All values are compile-time string literals (e.g., `"#FF3B30"`, `"#F2F2F7"`). These values flow through MudBlazor's `MudThemeProvider`, which applies them as CSS custom properties. There is no user input, external API call, or dynamic computation involved in color selection. The `Style="color: var(--mud-palette-text-secondary)"` replacement for `Color="Color.Secondary"` uses a CSS variable reference, not an inline computed value — this is the standard MudBlazor pattern and introduces no injection vector.
- **Affected artifacts:** design-output.yaml BUG-006 changes (ThemeRegistry.cs, ThemePreviewCard.razor, AppearancePage.razor)
- **Recommendation:** No action needed. Static hex literals in a theme registry have zero security attack surface. The CSS variable pattern (`var(--mud-palette-text-secondary)`) is safe as it references MudBlazor's own CSS custom properties.
- **Evidence:** All proposed color values in `design-output.yaml` are static hex strings. No `Environment.GetEnvironmentVariable`, `HttpClient` calls, or user-input-derived values are involved.

## Summary

This design review found **no security vulnerabilities** in the proposed UI bug fixes batch. All 8 bugs involve CSS/visual changes, UI navigation behavior fixes, form field type adjustments, and static color hex value updates. The innerHTML usage in `list-swipe-bridge.js` is safe (hardcoded SVG, no user input). The template picker exposes only game configuration DTOs (no PII). Navigation targets are hardcoded routes with server-controlled parameters. The skill rating null coercion preserves backend validation invariants. Theme color changes are compile-time literals. Five minor observations are documented for completeness, none requiring action before implementation proceeds.
