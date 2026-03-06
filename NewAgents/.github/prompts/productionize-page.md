---
name: Productionize
---

# Productionize Page Agent v2

Use this prompt when you need to refactor a Blazor page or feature into a
production-ready, mobile-first state.

---

## Usage

/productionize-page

yaml
Copy code

Or copy this prompt and specify the target page manually.

---

## Instructions

You are tasked with refactoring a Blazor page to **production-ready quality**.

### Tech Stack

- Blazor Hybrid
- .NET MAUI
- MudBlazor (Material Design)
- Entity Framework Core
- iOS / Android / Web

### Context

This app is near MVP. Many pages are partially implemented.
Refactor **this page only** into a production-ready, performant, mobile-first state.

---

## Priority Rules (Non-Negotiable)

When guidelines conflict, follow this order:

1. Correctness & data integrity
2. Apple Human Interface Guidelines (iOS-first interaction patterns)
3. Mobile usability
4. Existing project conventions (`CLAUDE.md`)
5. MudBlazor & Material Design defaults

Do **not** sacrifice correctness or Apple-style interaction patterns
for framework convenience.

---

## Core Goals

- Make the page complete, intuitive, and predictable
- Ensure fast performance on mobile devices
- Follow Apple interaction patterns for behavior and gestures
- Respect Material Design and MudBlazor visual conventions
- Avoid unnecessary complexity or visual noise
- Ensure observable behaviors are testable

---

## Checklist

### 📱 Layout & Navigation

- [ ] Prefer full-screen pages for mobile forms
- [ ] Avoid modal dialogs for data entry on mobile
- [ ] Dialogs may remain on desktop where appropriate
- [ ] Maintain consistent Back navigation
- [ ] After major actions:
  - **Create** → navigate to entity
  - **Generate** → navigate to result (e.g., bracket)
  - **Edit** → remain on page with confirmation feedback

---

### 🧭 User Guidance & Flow

- [ ] Apply progressive disclosure (hide advanced/optional controls by default)
- [ ] Make next actions visually clear
- [ ] Disable unavailable actions with explanations
- [ ] Show visual completion indicators for multi-step flows

---

### ⌨️ Forms & Keyboard Behavior

- [ ] Keyboard **Next** moves focus and scrolls field into view
- [ ] Keyboard **Done** closes keyboard by default
  - Submits only on simple, single-field forms
- [ ] Focused inputs must never be obscured by the keyboard
- [ ] Use scroll adjustment, not layout shifting
- [ ] Forms must:
  - Validate live
  - Enable submit immediately when valid
  - Show user-friendly inline validation
  - Never expose backend error messages directly

- [ ] Use built-in keyboard avoidance features to ensure the viewable area automatically adjusts when the keyboard is open on mobile (e.g., MAUI/Blazor Hybrid keyboard avoidance APIs)

---

### 🧭 Buttons & Primary Actions

**Primary action placement rules:**

- Mobile:
  - Use **either** a sticky bottom action **or**
  - A top-right navigation action
  - **Never both**
- Desktop:
  - Top-right or inline primary action

Additional rules:

- [ ] Use explicit, descriptive labels
- [ ] Remove duplicate or unused actions
- [ ] Avoid multiple competing primary actions

---

### 📋 List Interaction Patterns (Apple-First, Material-Respecting)

For list-based content (events, players, games, etc.):

#### Primary Interaction

- **Tap entire row** = Open / View details
- Do **not** show explicit “Open” buttons inside list rows
- Rows must remain visually calm and Material-consistent

#### Swipe Actions

- **Swipe left** reveals fast, common secondary actions
- Limit swipe actions to **2 (max 3)**:
  1. **Edit** (non-destructive)
  2. **Delete** (destructive, red, last)
- Swipe actions must not replace tap-to-open behavior

#### Long-Press (Hold) Actions

- **Long-press** opens a context menu when:
  - There are more than two secondary actions, or
  - Actions are advanced or infrequently used
- Long-press menus may include:
  - Duplicate
  - Archive
  - Share
  - Advanced configuration

#### Visibility Rules

- Avoid visible buttons inside list rows unless:
  - The action is non-navigational, and
  - The row supports multiple primary actions
- Destructive actions must always be hidden by default

> Gesture behavior should feel Apple-native while visual styling
> remains consistent with MudBlazor and Material Design.

---

### 🗑️ Destructive Actions (Lists & Pages)

- [ ] Destructive actions must require intentional input:
  - Swipe → Delete, or
  - Long-press → Delete
- [ ] Delete actions must:
  - Be visually destructive (red)
  - Appear last in menus or swipe actions
- [ ] Prefer **immediate action + Undo snackbar** when reversible
- [ ] Use confirmation sheets only when:
  - The action is irreversible, or
  - Undo would be unsafe (shared data, integrity loss)

---

### 🔄 Loading, Progress & Feedback

- [ ] Show progress indicators wherever users may wait
- [ ] Prefer inline loaders over full-page loaders
- [ ] Never block the UI without feedback

---

### ✅ Success, Errors & Messaging

- [ ] Inline confirmations for page-local actions ( no popup toasts)
- [ ] Snackbars with Undo for reversible actions
- [ ] Confirmation dialogs only for destructive or irreversible actions
- [ ] Avoid alert fatigue

---

### 🔍 Discoverability & Gesture Guidance

- Do **not** add instructional text for standard gestures:
  - Tap
  - Swipe
  - Long-press
- Prefer consistency over explicit instruction
- Use subtle, one-time hints only if absolutely necessary

---

### 🌍 Localization & Strings

- [ ] Extract all user-facing strings into resource files
- [ ] No hard-coded UI text
- [ ] Consistent terminology and tone

---

### 🏆 Domain Logic & Correctness

- [ ] Enforce scoring rules strictly
- [ ] Ensure brackets use correct point systems
- [ ] Event state must progress beyond Draft
- [ ] Support declaring winners
- [ ] Show Results / Victory screen when appropriate

---

### ⚡ Performance Best Practices

#### MAUI / Blazor Hybrid

- [ ] Avoid unnecessary re-renders
- [ ] Use `@key` for lists
- [ ] Split large pages into smaller components
- [ ] Avoid blocking the UI thread

#### Blazor

- [ ] Minimize cascading parameters
- [ ] Avoid unnecessary `StateHasChanged`
- [ ] Use `ShouldRender` where appropriate
- [ ] Prefer one-way data flow

#### MudBlazor

- [ ] Virtualize large lists (`MudVirtualize`)
- [ ] Avoid excessive dialogs or overlays
- [ ] Dispose dialogs properly

#### Entity Framework

- [ ] Use `AsNoTracking()` for read-only queries
- [ ] Avoid N+1 queries
- [ ] Load only required fields
- [ ] Ensure async DB access

---

### ♿ Accessibility & State

- [ ] Respect dark/light mode
- [ ] Ensure proper contrast
- [ ] Minimum tap targets: 44x44px
- [ ] Persist user preferences

---

## Testing Guidance

Focus on **observable behavior**, not third-party internals.

### Include tests for:

- Core services and business logic
- State transitions
- Conditional UI rendering
- Progressive disclosure
- Disabled actions and messaging

### Approach

- Unit tests for services and domain logic
- bUnit for Blazor components you control
- Mock EF Core access
- Cover success, failure, and edge cases
- Avoid testing MudBlazor internals

### Coverage Targets

- Critical logic/services: ~90%+
- Component/UI behavior: ~70–80%

If tests are not added, explicitly justify why in the plan document.

---

## Code Quality & Stability

- [ ] Remove dead code and unused dialogs
- [ ] Reuse shared components and logic
- [ ] Improve or maintain test coverage

---

## Do NOT

- Add unrelated features
- Change global architecture
- Over-animate
- Introduce unnecessary abstractions
- Override MudBlazor styling without justification

---

## Expected Outcome

After refactoring, this page should:

- Feel fast, intentional, and trustworthy
- Guide users naturally to the next step
- Prevent invalid or destructive actions
- Respect Apple interaction patterns and Material visuals
- Include maintainable test coverage
- Be safe to ship as-is

```bash
dotnet test tests/TourneyPal.Tests/TourneyPal.Tests.csproj
```
