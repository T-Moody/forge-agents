# Prompt: Complete Feature Workflow

Use this prompt for the complete end-to-end feature implementation workflow.

## Template

```
I want to implement the "{Feature Name}" feature for TourneyPal using the complete workflow.

## Feature Description
{Detailed description of the feature}

## Acceptance Criteria
- [ ] {Criterion 1}
- [ ] {Criterion 2}
- [ ] {Criterion 3}

## Workflow Steps
Please guide me through the complete feature workflow:

### Phase 1: Planning (@planner)
- Break down into action slices
- Identify dependencies
- Estimate complexity

### Phase 2: Documentation Setup (@docs)
- Create `docs/features/{feature}.md`
- Document acceptance criteria
- Define API contracts

### Phase 3: Implementation (@developer)
For each action slice (e.g., CreateEvent):
- Create {Action}Command.cs
- Create {Action}Validator.cs
- Implement {Action}Handler.cs
- Create {Action}Form.razor (UI)

### Phase 4: Tests (@tester)
- Write validator tests
- Write handler tests
- Write component tests (bUnit)

### Phase 5: Review (@reviewer)
- Review code quality
- Verify architecture compliance
- Fix any issues

### Phase 6: Documentation Update (@docs)
- Update feature documentation with implementation notes
- Document any deviations or decisions

After each phase, wait for my confirmation before proceeding to the next phase.

Reference agents:
- Planning: `.github/agents/planner.md`
- Development: `.github/agents/developer.md`
- Testing: `.github/agents/tester.md`
- Review: `.github/agents/reviewer.md`
- Documentation: `.github/agents/docs.md`
- UI: `.github/agents/ui-specialist.md`
```

## Example: Create Event Feature

```
I want to implement the "Create Event" feature for Tornimate using the complete workflow.

## Feature Description
Allow organizers to create new tournament events with a name, optional dates, and description. The system generates a unique join code that players can use to join the event.

## Acceptance Criteria
- [ ] Organizer can enter event name (required, max 100 chars)
- [ ] Organizer can set optional start and end dates
- [ ] Organizer can add a description
- [ ] System generates a unique 6-character join code
- [ ] Event is saved locally (offline support)
- [ ] Success message shown after creation
- [ ] Validation errors displayed inline

## Workflow Steps
Please start with Phase 1: Planning.

After you provide the plan, I'll confirm and we'll proceed to Phase 2.
```

---

## Quick Reference: Phase Prompts

### Start Phase 1 (Planning)

```
Let's start Phase 1: Planning for {Feature}.
Create an implementation plan with tasks, complexity, and dependencies.
```

### Start Phase 2 (Models)

```
Phase 1 complete. Let's start Phase 2: Models & Validation.
Create the DTOs and validators for {Feature}.
```

### Start Phase 3 (Handlers)

```
Phase 2 complete. Let's start Phase 3: Handlers.
Implement the command and query handlers for {Feature}.
```

### Start Phase 4 (Components)

```
Phase 3 complete. Let's start Phase 4: Components.
Create the UI components for {Feature} using MudBlazor.
```

### Start Phase 5 (Tests)

```
Phase 4 complete. Let's start Phase 5: Tests.
Write tests for validators, handlers, and components using bUnit.
```

### Start Phase 6 (Review)

```
Phase 5 complete. Let's start Phase 6: Review.
Review all code created for {Feature} and identify issues.
```

### Start Phase 7 (Documentation)

```
Phase 6 complete (all issues fixed). Let's start Phase 7: Documentation.
Create feature documentation at docs/features/{feature}.md.
```

### Complete Workflow

```
Phase 7 complete. Feature {Feature} is done!
Please provide a summary of what was implemented.
```
