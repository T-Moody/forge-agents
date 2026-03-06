# Prompt: Plan Feature Implementation

Use this prompt when starting work on a new feature.

## Template

```
I need to implement the "{Feature Name}" feature for TourneyPal.

## Feature Description
{Describe what the feature should do}

## Acceptance Criteria
- {Criterion 1}
- {Criterion 2}
- {Criterion 3}

## Context
- This is part of the {Event/Player/Game/Match/Standings/Scoreboard} feature
- Related features: {list related features}
- Offline support required: {yes/no}
- Tier: {Free/Premium/Organizer}

Please create an implementation plan following Action-Based Vertical Slice Architecture:
1. Identify the action slices needed (e.g., CreateEvent, UpdateEvent)
2. Break down each slice into tasks with complexity estimates
3. Identify dependencies between slices
4. Suggest implementation order
5. Note any technical considerations or risks

Reference the planner agent guidelines in `.github/agents/planner.md`.
```

## Example Usage

```
I need to implement the "Random Team Generator" feature for TourneyPal.

## Feature Description
Allow organizers to randomly generate balanced teams from the player list with configurable constraints.

## Acceptance Criteria
- Organizer can specify team size (2-8 players)
- Organizer can "pin" certain players to be on the same team
- Organizer can exclude certain player pairings
- Teams are generated randomly with even distribution
- Results can be regenerated until organizer is satisfied
- Generated teams are saved to the event

## Context
- This is part of the Team feature (new)
- Related features: Player (need player list), Event (teams belong to event)
- Offline support required: yes
- Tier: Free (basic), Premium (skill-aware balancing)

Please create an implementation plan following Action-Based Vertical Slice Architecture.
```
