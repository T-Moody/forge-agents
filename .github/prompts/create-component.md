# Prompt: Create Component

Use this prompt when creating a new Blazor component.

## Template

```
Create a Blazor component for the {Feature} feature in TourneyPal.

## Component Details
- **Name**: {ComponentName}.razor
- **Location**: 
  - Action-specific: `src/Tornimate.Shared/Features/{Feature}/{ActionName}/`
  - Shared: `src/Tornimate.Shared/Features/{Feature}/`
- **Type**: {Form / List / Card / Display / Page}

## Purpose
{What this component does}

## Props (Parameters)
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| {Prop} | {Type} | {Yes/No} | {Default} | {Description} |

## Events (EventCallbacks)
| Name | Type | Description |
|------|------|-------------|
| {Event} | {Type} | {Description} |

## UI Requirements
- {Layout requirement}
- {Styling requirement}
- {Responsiveness requirement}

## States to Handle
- [ ] Loading state
- [ ] Empty state
- [ ] Error state
- [ ] Success state

## Localization Keys Needed
- {Key}: {English text}

## MediatR Integration
- Commands: {List commands this component sends}
- Queries: {List queries this component uses}

Please create the component with:
1. `{ComponentName}.razor` - Markup only, no @code block
2. `{ComponentName}.razor.cs` - All C# logic in code-behind

Follow the patterns in:
- `.github/agents/ui-specialist.md`
- `.github/instructions/blazor.instructions.md`
```

## Example Usage

```
Create a Blazor component for the Event feature in TourneyPal.

## Component Details
- **Name**: EventCard.razor
- **Location**: `src/Tornimate.Shared/Features/Event/` (shared component at root)
- **Type**: Card

## Purpose
Display a summary card for a single event, showing key information and actions.

## Props (Parameters)
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| Event | EventDto | Yes | - | The event to display |
| ShowActions | bool | No | true | Whether to show action buttons |

## Events (EventCallbacks)
| Name | Type | Description |
|------|------|-------------|
| OnSelected | EventCallback<Guid> | Fired when card is clicked |
| OnEdit | EventCallback<Guid> | Fired when edit button clicked |
| OnDelete | EventCallback<Guid> | Fired when delete button clicked |

## UI Requirements
- Use MudCard with Elevation="2"
- Show event name prominently (h6)
- Show date in secondary text
- Show player count as a chip
- Show match status (X of Y complete)
- Action buttons: View, Edit, Delete

## States to Handle
- [x] Loading state - not applicable (data passed in)
- [ ] Empty state - not applicable
- [ ] Error state - not applicable
- [x] Success state - normal display

## Accessibility
- Card should be keyboard focusable
- Action buttons need aria-labels
- Delete should have confirmation

## MediatR Integration
- Commands: None (parent handles mutations)
- Queries: None (data passed via props)

Please create the component using MudBlazor.
```
