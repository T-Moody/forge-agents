# Task 10: Write Workflow Prompt

## Task Goal

Create the complete `feature-workflow.prompt.md` file in `NewAgentsAndPrompts/`, updating the workflow prompt to reinforce concurrency cap, per-task agent routing, and optional APPROVAL_MODE approval gates.

**Output file:** `NewAgentsAndPrompts/feature-workflow.prompt.md`

## depends_on

09

## Cluster

- **Cluster B (Agent Routing):** Must reinforce the `agent` field routing rule, consistent with orchestrator (Task 09) and planner (Task 07)
- **Cluster C (Approval Gates):** Must define `{{APPROVAL_MODE}}` variable, consistent with orchestrator (Task 09)

## In-Scope

- Update of `feature-workflow.prompt.md` following design.md specifications
- Frontmatter with `name: Feature Workflow` and `agent: orchestrator`
- Opening line: "You are the Orchestrator agent."
- **Rules** section (updated — adds 3 new rules):
  - Concurrency cap: max 4 concurrent per wave, sub-wave splitting for >4
  - Agent routing: tasks may specify `agent` field, dispatched accordingly
  - APPROVAL_MODE: conditional pausing after research synthesis and planning
- **Variables** section (NEW — documents {{USER_FEATURE}} and {{APPROVAL_MODE}} with defaults)
- Variable references at the end: {{USER_FEATURE}} and {{APPROVAL_MODE}}

## Out-of-Scope

- Modifying files in `.github/`
- Changing the fundamental prompt behavior (orchestrator remains the target agent)
- Defining detailed workflow steps (those are in the orchestrator agent file)

## Acceptance Criteria

1. File exists at `NewAgentsAndPrompts/feature-workflow.prompt.md` and is non-empty
2. Frontmatter has `name: Feature Workflow` and `agent: orchestrator`
3. Rules include concurrency cap: "Maximum 4 concurrent subagent invocations per wave" with sub-wave splitting mention (TS-4)
4. Rules include agent routing: "Tasks may specify an `agent` field" (TS-10)
5. Rules include APPROVAL_MODE: "If `{{APPROVAL_MODE}}` is `true`: pause for human approval after research synthesis and after planning" (TS-11)
6. Contains Variables section/table documenting both variables with defaults:
   - `{{USER_FEATURE}}`: required, no default
   - `{{APPROVAL_MODE}}`: optional, default `false`
7. Default behavior is fully autonomous (APPROVAL_MODE false or unset = no pauses) (TS-11)
8. Both `{{USER_FEATURE}}` and `{{APPROVAL_MODE}}` are referenced at the end as variable placeholders

## Test Requirements

1. **Concurrency test (TS-4):** Verify max 4 concurrent mentioned
2. **Agent routing test (TS-10):** Verify agent field routing rule present
3. **Approval gates test (TS-11):** Verify {{APPROVAL_MODE}} defined and documented
4. **Default autonomous test (TS-11):** Verify default is false/autonomous
5. **Cross-agent consistency:** Rules align with orchestrator Global Rules 7, 8, 9

## Implementation Steps

1. Read design.md § Per-Agent Design → 10. `feature-workflow.prompt.md` for target state and content specifications
2. Read design.md § Cross-Cutting Patterns for cluster consistency
3. Create `NewAgentsAndPrompts/feature-workflow.prompt.md` with:
   - Frontmatter (`name: Feature Workflow`, `agent: orchestrator`)
   - "You are the Orchestrator agent." opening
   - "Run the entire custom agent workflow end-to-end without stopping." instruction
   - `## Rules` section with ALL rules (existing + 3 new: concurrency, routing, approval)
   - `## Variables` section (table with USER_FEATURE and APPROVAL_MODE)
   - Variable placeholders at end:

     ```
     User Feature:
     {{USER_FEATURE}}

     Approval Mode:
     {{APPROVAL_MODE}}
     ```

## Estimated Effort

Low

## Completion Checklist

- [x] File created at `NewAgentsAndPrompts/feature-workflow.prompt.md`
- [x] Frontmatter correct (name, agent)
- [x] Concurrency cap rule (max 4, sub-wave splitting)
- [x] Agent routing rule (agent field in tasks)
- [x] APPROVAL_MODE rule (conditional pausing)
- [x] Variables section with defaults documented
- [x] Both variable placeholders at end of file
- [x] Default behavior is autonomous
- [x] Consistent with orchestrator Global Rules
