# Multi-Agent Workflow Guide

This guide describes how to use multiple Copilot agents together for effective feature development in TourneyPal.

## Project Structure Reference

```
TourneyPal.Common/Features/{Feature}/    # Shared contracts (DTOs, interfaces, commands, validators)
TourneyPal.Core/Features/{Feature}/      # Backend logic (handlers, repository, data service)
TourneyPal.UI/Features/{Feature}/        # Shared Razor components, client data service
TourneyPal/Features/{Feature}/           # API endpoints
tests/TourneyPal.Tests/Features/{Feature}/ # Tests
```

## Workflow Overview

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  @planner   │ -> │   @docs     │ -> │ @developer  │ -> │  @tester    │ -> │ @reviewer   │
│             │    │             │    │             │    │             │    │             │
│ Plan Tasks  │    │ Create Docs │    │ Implement   │    │ Write Tests │    │ Review Code │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
                                              │
                                              v
                                      ┌─────────────┐
                                      │@ui-specialist│
                                      │             │
                                      │  Polish UI  │
                                      └─────────────┘
```

## Phase 1: Planning (@planner)

**When**: Starting a new feature or significant change.

**Prompt**:

```
@planner I need to implement the "{Feature Name}" feature.

Description: {what it does}

Acceptance Criteria:
- {criterion 1}
- {criterion 2}

Please create an implementation plan with action slices.
```

**Output**: Implementation plan with tasks, complexity, and order.

**Action**: Review plan, ask clarifying questions, confirm before proceeding.

---

## Phase 2: Documentation Setup (@docs)

**When**: After planning is confirmed.

**Prompt**:

```
@docs Create initial documentation for the {Feature} feature based on this plan:

{paste plan from Phase 1}

Create `docs/features/{feature}.md` with:
- Overview
- User stories
- API contracts (placeholders)
- Acceptance criteria
```

**Output**: Feature documentation file created.

**Action**: Review docs, ensure alignment with requirements.

---

## Phase 3: Implementation (@developer)

**When**: For each action slice in the plan.

**Prompt**:

```
@developer Implement the {ActionName} action slice for {Feature}.

Files to create in TourneyPal.Common/Features/{Feature}/{ActionName}/:
- {Action}Command.cs (or {Action}Query.cs)
- {Action}Response.cs
- {Action}Validator.cs

Files to create in TourneyPal.Core/Features/{Feature}/{ActionName}/:
- {Action}Handler.cs

Files to create in TourneyPal.UI/Features/{Feature}/{ActionName}/:
- {Action}Dialog.razor
- {Action}Dialog.razor.cs

Business Logic:
{describe what it should do}

Validation:
{list validation rules}
```

**Output**: All files for the action slice.

**Action**: Review generated code, run to verify.

### UI Polish (@ui-specialist)

**When**: After basic implementation, if UI needs refinement.

**Prompt**:

```
@ui-specialist Review and improve the UI for {ActionName}Form.

Current implementation: {paste or reference file}

Improvements needed:
- Responsive layout
- Accessibility
- Loading/error states
- Localization
```

---

## Phase 4: Testing (@tester)

**When**: After each action slice is implemented.

**Prompt**:

```
@tester Write tests for the {ActionName} action slice.

Files to test:
- Features/{Feature}/{ActionName}/{Action}Validator.cs
- Features/{Feature}/{ActionName}/{Action}Handler.cs
- Features/{Feature}/{ActionName}/{Action}Form.razor

Test scenarios:
- {happy path}
- {error case}
- {edge case}
```

**Output**: Test files in `tests/TourneyPal.Tests/Features/{Feature}/{ActionName}/`

**Action**: Run tests, ensure all pass.

---

## Phase 5: Review (@reviewer)

**When**: After implementation and tests are complete.

**Prompt**:

```
@reviewer Review the {ActionName} action slice for {Feature}.

Files:
- Features/{Feature}/{ActionName}/*

Check for:
- Architecture compliance (Action-Based VSA)
- Code quality (var usage, nesting, method size)
- Blazor patterns (code-behind)
- Localization & Theming
- Error handling
- Test coverage
```

**Output**: Review report with issues and suggestions.

**Action**: Address critical issues, consider suggestions.

---

## Phase 6: Documentation Update (@docs)

**When**: After feature is complete and reviewed.

**Prompt**:

```
@docs Update documentation for {Feature} with implementation details.

Implemented slices:
- {ActionName}: {description}
- {ActionName}: {description}

Add:
- Actual API contracts (command/response)
- Validation rules
- Business rules discovered during implementation
- Testing notes
```

**Output**: Updated feature documentation.

---

## Quick Reference

| Phase      | Agent          | Input                | Output         |
| ---------- | -------------- | -------------------- | -------------- |
| Plan       | @planner       | Feature requirements | Task breakdown |
| Doc Setup  | @docs          | Plan                 | Initial docs   |
| Implement  | @developer     | Task details         | Code files     |
| UI Polish  | @ui-specialist | Component            | Improved UI    |
| Test       | @tester        | Implementation       | Test files     |
| Review     | @reviewer      | All files            | Review report  |
| Doc Update | @docs          | Implementation       | Final docs     |

## Tips

1. **Don't skip planning** - Even small features benefit from a quick plan
2. **Document early** - Creates clarity before implementation
3. **Test as you go** - Write tests after each action slice, not at the end
4. **Review incrementally** - Review after each slice, not the entire feature
5. **Update docs last** - Capture what was actually built, not just planned

## Example: Full Feature Workflow

```
User: @planner I need to implement "Add Player" for the Players feature.
      It should let organizers add players with name and optional avatar.
      Validation: Name required, max 50 chars, unique within event.

[Planner outputs plan]

User: @docs Create docs/features/players.md based on that plan.

[Docs creates initial documentation]

User: @developer Implement the CreatePlayer action slice.
      Location: TourneyPal.*/Features/Players/CreatePlayer/
      Validation: Name required, max 50 chars, unique per event.

[Developer creates all files across Common, Core, and UI]

User: @tester Write tests for CreatePlayer.

[Tester creates test files in tests/TourneyPal.Tests/Features/Players/CreatePlayer/]

User: @reviewer Review Features/Players/CreatePlayer/*

[Reviewer provides feedback]

User: @developer Fix the issues from the review: {list issues}

[Developer fixes issues]

User: @docs Update players.md with the final implementation details.

[Docs updates documentation]
```

## Agents Reference

| Agent          | File                              | Use For                |
| -------------- | --------------------------------- | ---------------------- |
| @planner       | `.github/agents/planner.md`       | Breaking down features |
| @developer     | `.github/agents/developer.md`     | Implementing code      |
| @tester        | `.github/agents/tester.md`        | Writing tests          |
| @reviewer      | `.github/agents/reviewer.md`      | Code review            |
| @ui-specialist | `.github/agents/ui-specialist.md` | UI/UX polish           |
| @docs          | `.github/agents/docs.md`          | Documentation          |

## Instructions Reference

| Topic        | File                                                | Content            |
| ------------ | --------------------------------------------------- | ------------------ |
| Architecture | `.github/instructions/architecture.instructions.md` | VSA structure      |
| Blazor       | `.github/instructions/blazor.instructions.md`       | Component patterns |
| C#           | `.github/instructions/csharp.instructions.md`       | Coding standards   |
| Testing      | `.github/instructions/testing.instructions.md`      | Test patterns      |
