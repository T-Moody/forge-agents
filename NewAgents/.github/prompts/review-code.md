# Prompt: Review Code

Use this prompt to get a code review for your changes.

## Template

```
Please review the following code changes for the {ActionName} action in the {Feature} feature of TourneyPal.

## Changed Files
- `Features/{Feature}/{ActionName}/{file1.cs}`
- `Features/{Feature}/{ActionName}/{file2.razor}`

## What Changed
{Brief description of the changes}

## Areas of Concern
- {Any specific areas you want reviewed}

Please review following the checklist in `.github/agents/reviewer.md`:
- Architecture alignment (Action-Based VSA, no Components/Models folders)
- Code quality (var usage, nesting, method size)
- Blazor patterns (code-behind, no @code blocks)
- Localization (no hardcoded strings)
- Theming (no hardcoded colors)
- Error handling
- Security
- Performance
- Testing
- Documentation
- Accessibility (for UI)
- Offline support

Provide feedback in the review output format specified in the reviewer agent.
```

## Example Usage

```
Please review the following code changes for the CreateEvent action in the Event feature of TourneyPal.

## Changed Files
- `Features/Event/CreateEvent/CreateEventCommand.cs`
- `Features/Event/CreateEvent/CreateEventValidator.cs`
- `Features/Event/CreateEvent/CreateEventHandler.cs`
- `Features/Event/CreateEvent/CreateEventForm.razor`
- `Features/Event/CreateEvent/CreateEventForm.razor.cs`

## What Changed
Implemented the Create Event action slice including:
- Command and response records
- FluentValidation validator
- MediatR handler with IDataService integration
- Form component with MudBlazor (code-behind pattern)

## Areas of Concern
- Is the error handling in the handler sufficient?
- Should the join code generation be extracted to a service?
- Is the form accessible?

Please provide a thorough code review.
```

---

# Prompt: Fix Review Issues

Use this prompt after receiving review feedback.

## Template

```
I received code review feedback for the {ActionName} action in the {Feature} feature. Please help me address these issues.

## Critical Issues
1. {Issue from review}
2. {Issue from review}

## Suggestions
1. {Suggestion from review}

## Files to Update
- `Features/{Feature}/{ActionName}/{file1.cs}`
- `{file2.razor}`

Please fix all critical issues and implement the suggested improvements.
```
