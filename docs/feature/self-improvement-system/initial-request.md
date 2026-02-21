# Initial Request: Self-Improvement System for Forge

## User Feature Request

Implement a structured cross-agent evaluation and post-mortem learning system so that agents can improve over time. This system is an ADDITIVE capability — it must NOT break existing workflows.

## Requirements

### PART 1 — Artifact Evaluation Capability

Modify all agents that CONSUME artifacts so they ALSO produce an evaluation section.

Example: If Designer reads feature.md from Spec Agent, Designer must output:

```yaml
artifact_evaluation:
  source_artifact: feature.md
  usefulness_score: (1-10)
  clarity_score: (1-10)
  useful_elements:
    - list specific sections that helped
  missing_information:
    - list concrete missing requirements
  information_not_used:
    - list information read, but not used
  inaccuracies:
    - list incorrect assumptions
  impact_on_work:
    - describe if rework or clarification was needed
```

This evaluation must be:

- Structured (YAML or JSON)
- Appended to agent output
- Stored alongside the artifact or in a dedicated evaluation directory

Do NOT allow free-form unstructured critique.

### PART 2 — Orchestrator Telemetry Tracking

Extend the orchestrator to capture execution metadata:

For each agent run track:

- agent_name
- execution_time_ms
- retry_count
- failure_reason (if any)
- number_of_iterations
- human_intervention_required (true/false)

Store this in a structured run log.

### PART 3 — Post-Mortem Agent

Create a new agent called "PostMortemAgent" that runs AFTER the workflow completes.

Its responsibilities:

1. Collect all artifact evaluations
2. Collect orchestrator telemetry logs
3. Identify patterns such as:
   - Frequently missing info from certain agents
   - Recurring inaccuracies
   - Bottleneck agents
   - High retry rates
4. Produce a structured report:

```yaml
post_mortem_report:
  recurring_issues:
  bottlenecks:
  most_common_missing_information:
  agent_accuracy_scores:
  improvement_recommendations:
```

The output must be deterministic and structured.

### PART 4 — Storage Design

Implement a persistent storage location for:

- /agent-metrics/
- /artifact-evaluations/
- /post-mortems/

Ensure:

- Runs are timestamped
- Data is append-only
- No existing files are overwritten

### PART 5 — Safety Requirements

DO NOT:

- Change existing agent behavior
- Modify artifact formats
- Introduce breaking changes
- Add heavy dependencies

This must be an additive capability.

### PART 6 — Deliverables

After implementation, produce:

1. A summary of changes made
2. Where evaluation data is stored
3. How agents now generate evaluations
4. How to trigger the post-mortem agent

## Additional Requirement — Orchestrator Tool Restriction

Update the orchestrator agent so that it only has the ability to use these tools: `[agent, agent/runSubagent, memory]`. The goal is that the orchestrator only orchestrates because it doesn't have tools to do anything else. Update the orchestrator markdown since there are instructions to make file edits — the orchestrator will have to spawn subagents for that work.

## Constraints

- This is an existing multi-agent orchestration system called "Forge"
- Agents are defined as `.agent.md` files in `.github/agents/`
- Artifacts are stored under `docs/feature/<feature-slug>/`
- Memory system: `memory.md` (shared) + `memory/<agent>.mem.md` (isolated per agent)
- Must be additive — no breaking changes to existing agents or workflows
