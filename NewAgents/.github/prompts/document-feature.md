# Prompt: Document Feature

Use this prompt when documenting a completed feature.

## Template

```
Create documentation for the {Feature} feature in TourneyPal.

## Feature Overview
{Brief description of what the feature does}

## Action Slices Implemented
- `{ActionName}/` - {purpose}
  - Command: {Command}
  - Handler: {Handler}
  - UI: {Form/Component}
- `{ActionName}/` - {purpose}
  - ...

## Shared Components (at feature root)
- `{Feature}Card.razor` - {purpose}
- `{Feature}Dto.cs` - {purpose}

## Business Rules
1. {Rule 1}
2. {Rule 2}

## Tier
- Free: {what's included}
- Premium: {what's included}
- Organizer: {what's included}

## Offline Behavior
{How this feature works offline}

Please create the feature documentation file at `docs/features/{feature-name}.md` following the template in `.github/agents/docs.md`.

Include:
- Overview
- User stories
- Action slice documentation (command/response, validation rules)
- Component documentation with props/events
- Business rules
- Tier breakdown
- Offline behavior
- Testing notes
```

## Example Usage

```
Create documentation for the Event feature in TourneyPal.

## Feature Overview
Allow organizers to create and manage tournament events, including setting dates, descriptions, and generating shareable join codes.

## Action Slices Implemented
- `CreateEvent/` - Creates a new event
  - Command: CreateEventCommand
  - Handler: CreateEventHandler
  - UI: CreateEventForm.razor
- `UpdateEvent/` - Updates event details
  - Command: UpdateEventCommand
  - Handler: UpdateEventHandler
  - UI: UpdateEventForm.razor
- `DeleteEvent/` - Deletes an event
  - Command: DeleteEventCommand
  - Handler: DeleteEventHandler

## Shared Components (at feature root)
- `EventCard.razor` - Display card for event summary
- `EventDto.cs` - Data transfer object for events

## Business Rules
1. Event names must be unique within organizer's events
2. Join codes are 6 alphanumeric characters, auto-generated
3. Events can optionally have start/end dates
4. Deleting an event deletes all associated data (games, players, matches)

## Tier
- Free: Basic event creation, 128 player limit
- Premium: Multi-day events, advanced settings
- Organizer: Cloud sync, shareable links

## Offline Behavior
- Events are stored locally in IndexedDB (web) or SQLite (MAUI)
- Join codes work for local event access
- Cloud sync (Organizer tier) uses last-write-wins for conflicts

Please create the full feature documentation.
```
