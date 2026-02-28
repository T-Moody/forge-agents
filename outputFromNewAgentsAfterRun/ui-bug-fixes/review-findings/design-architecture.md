# Adversarial Review: design — claude-opus-4.6

## Review Metadata

- **Review Focus:** architecture
- **Risk Level:** 🟡
- **Round:** 1
- **Run ID:** 2026-02-26T14:30:00Z

## Overall Verdict

**Verdict:** needs_revision
**Confidence:** High

## Findings

### Finding 1: Design violates its own acceptance criterion AC-006-8 regarding dark mode background

- **Severity:** Critical
- **Category:** design
- **Description:** The spec acceptance criterion AC-006-8 explicitly states: "dark mode background is not pure black." However, the design (D-6) proposes changing Professional theme `PaletteDark.Background` from `#1a1a27` to `#000000` (pure black) and `BackgroundGray` from `#151521` to `#000000`. This is a direct contradiction between the design and the specification it claims to satisfy. The existing test `AllNonHcThemes_PaletteDark_BackgroundIsNotPureBlack` (ThemeRegistryTests.cs line 224) encodes this invariant for all non-high-contrast themes. The design's acceptance criteria mapping at the bottom of design.md marks AC-006-8 as "Covered" with no qualification.
- **Affected artifacts:** design-output.yaml (D-6, Professional PaletteDark changes), spec-output.yaml (AC-006-8), ThemeRegistryTests.cs line 220-228
- **Recommendation:** Either (a) revise the dark mode Background to a near-black value (e.g., `#1C1C1E` which is Apple's elevated surface, or `#0A0A0A`) instead of pure `#000000` to satisfy AC-006-8 and the existing invariant, or (b) formally request a spec amendment to change AC-006-8 if the intent is to adopt Apple's pure-black OLED dark mode. The test invariant must be explicitly reconciled — the design should state whether the invariant is being preserved or deliberately changed, not silently broken.
- **Evidence:** AC-006-8 in spec-output.yaml line 373: `"ThemeRegistryTests still pass — all 22 core properties are non-null, background hierarchy is maintained, dark mode background is not pure black."` Design-output.yaml Professional PaletteDark changes: `Background: { from: "#1a1a27", to: "#000000" }`, `BackgroundGray: { from: "#151521", to: "#000000" }`. Test assertion at ThemeRegistryTests.cs line 226: `Assert.NotEqual("#000000", theme.Theme.PaletteDark.Background.Value, StringComparer.OrdinalIgnoreCase)`.

### Finding 2: PaletteLight Surface = #FFFFFF breaks established theme system invariant

- **Severity:** Major
- **Category:** design
- **Description:** The design proposes changing Professional theme `PaletteLight.Surface` from `#F2F2F6` to `#FFFFFF`. The test `AllNonHcThemes_PaletteLight_Has22CoreProperties` (ThemeRegistryTests.cs line 202) asserts `AssertPaletteColorNotEqual("#FFFFFF", palette.Surface)` for ALL non-high-contrast themes. This invariant encodes a deliberate architectural decision: Surface (elevated content areas) should be tinted, not pure white, to provide visual depth — pure white is reserved for the high-contrast theme's accessibility mode. The design's testing section mentions "5-10 tests in ThemeRegistryTests.cs (hex value assertions)" but does not distinguish between simple value assertions and structural invariants. An implementer following the design would need to either (a) remove the Surface ≠ #FFFFFF invariant, weakening the theme system's quality gate, or (b) use a near-white value (e.g., `#FAFAFA` or `#FEFEFE`).
- **Affected artifacts:** design-output.yaml (D-6, Professional PaletteLight Surface change), ThemeRegistryTests.cs lines 198-204
- **Recommendation:** Either use a near-white Surface value (e.g., `#FAFAFA`) that satisfies Apple HIG "white card on gray background" intent without being pure `#FFFFFF`, or explicitly document in the design that the Surface ≠ #FFFFFF invariant is being revoked with rationale. Additionally, the existing comment "Surface should be darker" in `ProfessionalTheme_PaletteLight_BackgroundHierarchyIsCorrect` is semantically violated since Surface (#FFFFFF) would now be lighter than Background (#F2F2F7), inverting the hierarchy — even though the assertion itself only checks inequality.
- **Evidence:** ThemeRegistryTests.cs line 202: `AssertPaletteColorNotEqual("#FFFFFF", palette.Surface)`. Line 200 comment: `// Assert — Surface hierarchy (R1.6): Surface ≠ #FFFFFF, DrawerBackground == Surface`. Design-output.yaml: `Surface: { from: "#F2F2F6", to: "#FFFFFF" }`.

### Finding 3: DrawerBackground omitted from Professional light palette changes, breaking DrawerBackground == Surface coupling

- **Severity:** Major
- **Category:** design
- **Description:** The design changes Professional `PaletteLight.Surface` from `#F2F2F6` to `#FFFFFF` but does not include `DrawerBackground` in the change list. Currently `DrawerBackground = "#F2F2F6"` (ThemeRegistry.cs line 79), which equals Surface. The test invariant `Assert.Equal(palette.Surface.Value, palette.DrawerBackground.Value)` at ThemeRegistryTests.cs line 203 enforces this coupling. After the design's proposed Surface change, Surface would be `#FFFFFF` while DrawerBackground remains `#F2F2F6`, breaking the invariant. This represents an incomplete impact analysis of the property coupling graph within ThemeRegistry. The design's change list for BUG-006 Professional PaletteLight enumerates 10 properties (Primary through Error) but misses DrawerBackground as a property that must change in tandem with Surface.
- **Affected artifacts:** design-output.yaml (D-6, Professional PaletteLight changes — missing DrawerBackground), ThemeRegistry.cs line 79, ThemeRegistryTests.cs lines 203-204
- **Recommendation:** Add `DrawerBackground: { from: "#F2F2F6", to: "#FFFFFF" }` (or whatever Surface's final value is) to the Professional PaletteLight change list. Audit all other themes for similar Surface/DrawerBackground coupling to ensure they remain synchronized. If the relationship is intentionally being decoupled, document the rationale and update the test invariant.
- **Evidence:** ThemeRegistry.cs line 79: `DrawerBackground = "#F2F2F6"`. ThemeRegistryTests.cs line 185: `Assert.Equal(palette.Surface.Value, palette.DrawerBackground.Value, StringComparer.OrdinalIgnoreCase)`. Design-output.yaml Professional PaletteLight changes list does not include DrawerBackground.

### Finding 4: Inline Style pattern for TextSecondary color should be documented as established convention

- **Severity:** Minor
- **Category:** maintainability
- **Description:** BUG-007 and BUG-006 replace `Color="Color.Secondary"` with `Style="color: var(--mud-palette-text-secondary)"` on MudText components. This is architecturally correct — MudBlazor's `Color` enum has no `TextSecondary` value, so inline `Style` referencing CSS custom properties is the only viable approach. However, the design introduces this as a fix for 6 instances now with 60+ remaining instances documented for future work. This establishes an inline-style pattern that bypasses MudBlazor's component API as the canonical approach for muted text. The design should acknowledge this as a new codebase convention (not just a one-off fix) to prevent future developers from re-introducing `Color.Secondary` for muted text intent.
- **Affected artifacts:** design-output.yaml (D-7), design.md §4.7
- **Recommendation:** Document `Style="color: var(--mud-palette-text-secondary)"` as the standard pattern for muted text in the relevant instruction file (e.g., blazor.instructions.md or components.instructions.md) alongside the existing "no hardcoded colors" rule. This ensures the remaining 60+ instances are fixed consistently and the pattern isn't re-introduced by future contributors.
- **Evidence:** BUG-007 audit in design-output.yaml shows 99 occurrences across 33 files, with 65 identified as "muted text intent." Only 6 are fixed in this batch. No instruction file update is proposed.

## Summary

The design is architecturally sound in its overall approach: vertical slice boundaries are preserved, component reuse is appropriate (TemplatePickerDialog within its own slice), the BUG-004 cancel-vs-navigate decoupling is the correct pattern (matching ParticipantsTabContent), and implementation wave ordering correctly handles file-level dependencies. However, the BUG-006 theme palette redesign has a critical spec-design contradiction (AC-006-8 says "dark mode background is not pure black" but D-6 proposes Background = #000000), two broken test invariants (Surface ≠ #FFFFFF, DrawerBackground == Surface coupling), and an incomplete property change list. These must be reconciled before implementation to avoid cascading test failures and inadvertent weakening of the theme system's quality gates.
