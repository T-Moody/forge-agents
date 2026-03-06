# Prompt: Write Tests

Use this prompt when writing tests for an action slice.

## Template

```
Write tests for the {ActionName} action slice in the {Feature} feature of TourneyPal.

## Test Scope
- [ ] Validator tests
- [ ] Handler tests
- [ ] Component tests (bUnit)

## Target Files (in Features/{Feature}/{ActionName}/)
- Validator: `{Action}Validator.cs`
- Handler: `{Action}Handler.cs`
- Component: `{Action}Form.razor`

## Test Requirements

### Validator Tests
Test these scenarios:
- {Valid input scenario}
- {Invalid input scenario 1}
- {Invalid input scenario 2}
- {Edge case}

### Handler Tests
Test these scenarios:
- {Happy path}
- {Error case 1}
- {Error case 2}
- {Edge case}

### Component Tests
Test these scenarios:
- {Renders correctly with valid data}
- {Handles user interaction}
- {Shows loading state}
- {Shows error state}

## Dependencies to Mock
- `IDataService`
- `IMediator`
- {Other dependencies}

Please create the test files following the patterns in:
- `.github/agents/tester.md`
- `.github/instructions/testing.instructions.md`

Location: `tests/Tornimate.Shared.Tests/Features/{Feature}/{ActionName}/`
```

## Example Usage

```
Write tests for the CreateEvent action slice in the Event feature of TourneyPal.

## Test Scope
- [x] Validator tests
- [x] Handler tests
- [x] Component tests (bUnit)

## Target Files (in Features/Event/CreateEvent/)
- Validator: `CreateEventValidator.cs`
- Handler: `CreateEventHandler.cs`
- Component: `CreateEventForm.razor`

## Test Requirements

### Validator Tests (CreateEventValidatorTests.cs)
Test these scenarios:
- Valid: Name provided, dates valid → passes
- Invalid: Empty name → fails with "Name is required"
- Invalid: Name > 100 chars → fails with length message
- Invalid: EndDate before StartDate → fails
- Edge: StartDate = today → passes

### Handler Tests (CreateEventHandlerTests.cs)
Test these scenarios:
- Happy path: Valid command → creates event, returns success with ID
- Happy path: Generates unique 6-char join code
- Error: DataService throws → returns failure response
- Error: Logs error on failure

### Component Tests (EventCardTests.cs)
Test these scenarios:
- Renders event name in h6
- Renders date when provided
- Shows "No date set" when date is null
- Click card → invokes OnSelected with event ID
- Edit button → invokes OnEdit
- Delete button with confirmation → invokes OnDelete
- Keyboard: Enter key on card → invokes OnSelected

## Dependencies to Mock
- `IDataService` (for handlers)
- `ILogger<T>` (for handlers)
- `IMediator` (for components if needed)
- `ISnackbar` (for components)

Please create all test files.
```
