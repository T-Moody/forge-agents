# Prompt: Implement Action Slice

Use this prompt when implementing a new action slice (command/query + handler + validator + UI).

## Template

````
Implement the {ActionName} action slice for the {Feature} feature in TourneyPal.

## Action Details
- **Name**: {ActionName} (e.g., CreateEvent)
- **Type**: {Command (mutates state) / Query (reads state)}
- **Location**: `src/Tornimate.Shared/Features/{Feature}/{ActionName}/`

## Command/Query
```csharp
public record {Action}Command(
    {PropertyType} {PropertyName},
    // Add properties
) : IRequest<{Action}Response>;
````

## Expected Response

```csharp
public record {Action}Response(
    {PropertyType} {PropertyName}
);
```

## Business Logic

1. {Step 1}
2. {Step 2}
3. {Step 3}

## Validation Rules

- {Property}: {Rule}

## Error Cases

- {Error case 1}: {Expected behavior}
- {Error case 2}: {Expected behavior}

## UI Requirements

- {UI requirement 1}
- {UI requirement 2}

Please implement the complete action slice:

1. `{Action}Command.cs` - The command/query record
2. `{Action}Response.cs` - The response DTO
3. `{Action}Validator.cs` - FluentValidation validator
4. `{Action}Handler.cs` - The MediatR handler
5. `{Action}Form.razor` - The UI component (if applicable)
6. `{Action}Form.razor.cs` - The code-behind

All files go in: `Features/{Feature}/{ActionName}/`

Follow the patterns in `.github/agents/developer.md` and `.github/instructions/`.

```

## Example Usage

```

Implement the CreateEvent action slice for the Event feature in TourneyPal.

## Action Details

- **Name**: CreateEvent
- **Type**: Command (mutates state)
- **Location**: `src/Tornimate.Shared/Features/Event/CreateEvent/`

## Command

```csharp
public record CreateEventCommand(
    string Name,
    DateTime? StartDate,
    DateTime? EndDate,
    string? Description
) : IRequest<CreateEventResponse>;
```

## Expected Response

```csharp
public record CreateEventResponse(
    Guid Id,
    bool Success,
    string? ErrorMessage = null
);
```

## Business Logic

1. Validate the command
2. Create a new Event entity with a unique ID
3. Save to data store
4. Return success with the new ID

## Validation Rules

- Name: Required, max 100 characters
- StartDate: Must be in the future (if provided)
- EndDate: Must be after StartDate (if both provided)

## Error Cases

- Duplicate name: Return failure with message
- Data service error: Return failure, log error

## UI Requirements

- Form with name, dates, description fields
- Submit button with loading state
- Success notification on create
- Error display on failure

Please implement the complete action slice.

```


```csharp
public record CreateEventResponse(
    Guid EventId,
    string JoinCode,
    bool Success,
    string? ErrorMessage = null
);
```

## Business Logic

1. Validate the command
2. Generate a unique 6-character join code
3. Create the Event entity with a new GUID
4. Persist via IDataService
5. Return the response with event ID and join code

## Validation Rules

- Name: Required, max 100 characters
- StartDate: If provided, must be today or future
- EndDate: If provided, must be >= StartDate

## Error Cases

- Empty name: Return validation error
- Database failure: Return failure response with error message

Please implement all files following the developer agent patterns.

```

```
